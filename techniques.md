# Techniques — how to get at any of this

The methods behind the findings in the other guides. Recorded separately because
they outlive any particular result: the same handful of tricks answered questions
about firmware packaging, container platform emulation, and the camera↔controller
protocol.

Ordered roughly by how much time each one saves.

---

## 1. Make the controller log its own plaintext

**The single highest-value technique.** `:7442` is TLS, so a packet capture gives
ciphertext. You do not need to break it — the daemon will hand you the plaintext.

`:7442` is served by **`ds`**, a Rust binary (`tokio-tungstenite`,
`src/services/device_proxy/connection_handler.rs`). Its config has payload
tracing, switched off by default:

```jsonc
// /usr/share/ds/ds.json
"trace": {
  "log_websocket_payload": false,    // -> true
  "dump_websocket_payload": false,   // -> true
  "log_smartdetect_payload": false,
  "log_smartmotion_payload": false,
  "auto_dump_websocket_threshold": 50
}
```

```bash
cp /usr/share/ds/ds.json /usr/share/ds/ds.json.orig
# set the two flags true
systemctl restart ds
```

Then:

| Flag | Output | Character |
|---|---|---|
| `log_websocket_payload` | `/srv/ds/logs/ds.log` — hex+ASCII dumps of every frame | **Everything.** Needs reassembly (§2) |
| `dump_websocket_payload` | `/srv/ds/logs/payloads.jsonl` — one JSON object per line | Clean, but **event-gated**; captured only `EventSmartDetect` in practice |

`payloads.jsonl` lines look like:

```json
{"timestamp":"…","offset_ms":0,"connection_id":"<MAC>-<epoch>",
 "mac_address":"<MAC>","function_name":"EventSmartDetect","payload":{…}}
```

### Why the obvious alternatives don't work

- **`LD_PRELOAD` on `SSL_read`/`SSL_write`:** `ds` links **no libssl**
  (`ldd $(which ds)` shows nothing SSL-ish). No symbols to hook.
- **gdb:** same problem, plus it's a stripped Rust binary.
- **`SSLKEYLOGFILE`:** not honoured; nothing in `strings` suggests support.
- **TLS MITM:** *would* work — the camera does not validate the controller's
  certificate, so a terminating proxy on `:7442` forwarding to `ds` (via an
  iptables `REDIRECT`) is viable. Keep it in reserve; it's strictly more work than
  flipping two booleans.

**Remember to revert** `ds.json` afterwards. The dumps grow quickly and contain
device identifiers.

---

## 2. Reassemble hex+ASCII log dumps into JSON

`log_websocket_payload` emits frames as offset-prefixed hex, one line per 16
bytes. Group by offset restarting at `0`, concatenate, then find the first `{`.

```python
import re, json
rx = re.compile(r'\s([0-9A-F]{1,4})\|((?: [0-9A-F]{2})+)')
cur, frames = bytearray(), []
for ln in open('ds.log', errors='replace'):
    m = rx.search(ln)
    if not m:
        continue
    off = int(m.group(1), 16)
    if off == 0 and cur:                 # new frame starts
        frames.append(bytes(cur)); cur = bytearray()
    cur.extend(int(b, 16) for b in m.group(2).split())
if cur: frames.append(bytes(cur))

env = []
for fb in frames:
    s = fb.decode('utf8', 'replace'); i = s.find('{')
    if i >= 0:
        try: env.append(json.loads(s[i:]))
        except Exception: pass
```

263 envelopes came out of a 17 000-line log this way. **Direction is free**:
`from == "ubnt_avclient"` is the camera, `from == "UniFiVideo"` the controller —
so you can count each verb per direction and see the ack pattern immediately.

---

## 3. Parse a pcap without tshark

`:7550` (media) is **plain TCP**, so it is readable directly. If you don't want to
install tshark, the pcap format is trivial: 24-byte global header, then per-packet
16-byte header + raw frame.

```python
import struct, collections
f = open('cap.pcap','rb'); f.read(24)
streams = collections.defaultdict(bytearray)
while (ph := f.read(16)) and len(ph) == 16:
    _,_,cl,_ = struct.unpack('<IIII', ph)
    pkt = f.read(cl)
    if len(pkt) < 34 or struct.unpack('!H', pkt[12:14])[0] != 0x0800:
        continue                                   # not IPv4
    ihl = (pkt[14] & 0x0f) * 4
    if pkt[14+9] != 6:                             # not TCP
        continue
    t = 14 + ihl
    sp, dp = struct.unpack('!HH', pkt[t:t+4])
    doff = (pkt[t+12] >> 4) * 4
    if dp == 7550:                                 # camera -> controller
        streams[sp].extend(pkt[t+doff:])
```

Keying on the source port separates concurrent tracks. Expect **connections that
started before the capture to have no FLV header** — only the ones that opened
during the capture are parseable from the beginning, which is why forcing a
re-adoption (§5) matters.

---

## 4. Parse `extendedFlv`, and know what standard tools do wrong

Ubiquiti's container is FLV with **20 bytes between tags where standard FLV has
4** — a 4-byte previous-tag-size plus a 16-byte wall-clock trailer — and a flags
byte of `0x07` rather than `0x05`.

Walking tags therefore advances by `11 + data_size + 4 + 16`:

```python
p = data.find(b'FLV') + 9 + 4          # header + PreviousTagSize0
while p + 11 <= len(data):
    tt = data[p]                                    # 8 audio, 9 video, 18 script
    ds = int.from_bytes(data[p+1:p+4], 'big')
    body = data[p+11:p+11+ds]
    ...
    p += 11 + ds + 4 + 16                           # note the 16
```

**Counting tag types is the whole trick** for settling codec and audio questions:
`tt == 8` is audio, `tt == 9` video, `tt == 18` script. Per-tag first bytes tell
you the rest — `body[0] >> 4` is the frame type, `body[0] & 0x0f` the codec id.

⚠️ **Converting to standard FLV and handing it to ffmpeg is unreliable here.**
The documented conversion (force flags to `0x05`, recompute previous-tag-size,
drop the 16-byte trailer) produces a byte-clean file, but ffmpeg still
mis-detects it — inventing a dozen phantom streams — because **Ubiquiti uses
video codec id `8` for H.265** and no standard demuxer maps that. Trust your own
tag walk over ffprobe's opinion.

For audio, the header nibbles lie: `0xaf` decodes as "44 kHz stereo", but for AAC
those bits are ignored and the truth is in the AudioSpecificConfig two bytes later
(`objectType=2`, `sampleRate=16000`, `channels=1`).

---

## 5. Force a fresh handshake

The adoption sequence only appears when a camera connects. To replay it on demand,
restart the controller side and let the camera reconnect:

```bash
docker exec <c> systemctl restart ds unifi-protect
# camera reconnects in ~60-80s; watch for it:
grep -c 'RECEIVED ubnt_avclient_hello' /srv/unifi-protect/logs/cameras.avclient.log
```

**Enable payload logging *before* restarting**, or you capture the steady state
and miss the handshake. Count hellos before and after to confirm you got a fresh
one rather than a reconnect of an existing session.

---

## 6. Dump the real console's full package inventory

Settles any "what does a real UNVR actually run?" question in one look, and it is
how the dropped-dependency bug was found.

`fwextract.sh` (this project's fork) takes `FLOCK_KEEP_SCRATCH=1`, which preserves
the extracted squashfs and writes:

```
<scratch>/all-installed-packages.txt      # 431 entries: package version | Maintainer
```

Generated with:

```bash
dpkg-query --admindir=<squashfs>/var/lib/dpkg/ -W -f='${Package} ${Version} | ${Maintainer}\n'
```

**Never filter this by maintainer.** Ubiquiti is inconsistent about the field —
`ai-feature-console` is maintained by `unifi-protect-ai`, `ds`/`ms`/`msr`/`msp`/
`mst`/`msf` by a bare `Ubiquiti`, `unifi-rclone` by `uos-fw-dev`,
`ubnt-opencv4-libs` by `admin@opencv.org`. Filtering on `@ubnt.com|@ui.com` drops
ten hard dependencies of `unifi-protect`.

### Recovering an individual package from a preserved squashfs

```bash
dpkg-repack --root=<squashfs>/ --arch=arm64 <package>
```

No re-download needed, which matters when the firmware is 0.74 GB.

---

## 7. Read config and vocabulary out of the binaries

`strings` on the Ubiquiti daemons is unreasonably productive. `ds` alone yields:

- **Its whole config schema** — `struct TraceConfig` with the field names that
  became §1, plus `WebSocketConfig`, `SslConfig`, `EventProcessingConfig`.
- **Every environment variable it honours** — `LOG_LEVEL`,
  `SSL_CERTIFICATE_VERIFICATION_ENABLED`, `SSL_ALLOW_SELF_SIGNED`,
  `SSL_IGNORE_SNI_HOSTNAME`, `DS_RECORDING_DISABLED`, `WEBSOCKET_*` tunables.
  These can be set via `EnvironmentFile=-/etc/ds/default`, which does not exist by
  default and is yours to create.
- **Protocol vocabulary** — `struct CameraMessage {functionName, inResponseTo,
  messageId, timeStamp, …}`, and a `CameraFeatureFlags` struct listing scores of
  capability names (`hasLiveviewTracking`, `hasLineCrossingCounting`,
  `presetTour`, `hasSmartZoom`, `streamEncryptable`, `supportFullHdSnapshot`, …).
- **Rust source paths**, which reveal the module layout —
  `src/services/device_proxy/connection_handler.rs`, `src/ds/device_service.rs`.

For the camera-side binaries the same applies, with the caveat from
[`unifi-camera-reference.md`](unifi-camera-reference.md) §1: there is no
`strings(1)` on the camera, so pull binaries off base64-encoded over SSH and
analyse them elsewhere.

**Look for a config struct before you reach for a debugger.** Twice now the thing
we wanted was an off-by-default flag.

---

## 8. Know which log holds what

Reading the right file is often the whole task. On the controller:

| Path | Contents |
|---|---|
| `/srv/unifi-protect/logs/cameras.avclient.log` | **The `:7442` channel.** `RECEIVED`/`SENT <functionName>` with message ids, and payloads for IPC-style verbs. The adoption sequence in order |
| `/srv/unifi-protect/logs/cameras.log` | Per-camera lifecycle, and **the full ingest configuration** pushed to each channel — codec, `avSerializer`, destinations |
| `/srv/unifi-protect/logs/devices.websockets.log` | Connection establishment: `wsproto=`, `isAvClient=`, fingerprint verification, reported firmware |
| `/srv/unifi-protect/logs/cameras.ptz.log` | PTZ channel specifically — rejections, return-home, disconnect codes |
| `/srv/ds/logs/ds.log` | `ds`'s own log; **frame dumps land here** when §1 is enabled |
| `/srv/ds/logs/payloads.jsonl` | Structured envelope dump (event-gated) |
| `/srv/unifi-protect/logs/nvr.storage.log` | Storage gRPC health — a code-14 retry loop here means the `ustate` chain is broken |
| `/data/unifi-core/logs/http.log` | Every HTTP request with status — `403` on `/api/auth/login` means the *username* is wrong |
| `/data/unifi-core/logs/auth.log` | Attempted username and real rejection reason |
| `/srv/unifi-protect/downloads/` | Camera firmware `.bin`s the controller has fetched — evidence it manages camera firmware |

---

## 9. Test a dangerous package without risking the host

`ustd` ships `uscsi-reset-wipe.service` (a "Reset-format Monitor",
`WantedBy=local-fs-pre.target`) and udev rules that fire on block-device add. To
find out what it does before letting it near a disk that matters:

**Containment that actually contains:** run a throwaway container with **no
`--privileged` and no `--device`**, so the kernel simply never exposes block
devices to it.

```bash
docker run -d --name test --cap-add SYS_ADMIN --cap-add SYS_RESOURCE \
  --tmpfs /run --tmpfs /run/lock --tmpfs /tmp <image>
docker exec test ls /dev/sd\*     # must fail — that's the proof
```

Caveat: systemd will **not** come up this way — `/sys/fs/cgroup` is mounted
read-only without privileges, so PID 1 stalls with no bus. Work around it by
running the daemons **by hand** instead of via systemd:

```bash
docker exec -d test /usr/sbin/usdbd -j        # from the unit's ExecStart
docker exec test ls -l /run/ustd.sock         # did it create its socket?
docker exec -d test ustate daemon start
```

That was enough to prove the whole chain before touching the real container.

### Then mask before installing

`dpkg` runs `systemctl preset`, which will happily *enable* a service you don't
want. Masking first blocks that — and masking works on units that don't exist yet:

```bash
systemctl mask uscsi-reset-wipe.service uscsi-rescan.service uscsi-disk-ready.service
apt-get install -y ./ustd_*.deb
# installer logs: "Failed to preset unit: Unit file ... is masked"  <- what you want
```

Verify afterwards with `blkid` and a file listing that the disk is untouched.

---

## 10. Verify storage claims by moving bytes, not by asking

`docker info` will report the `Docker Root Dir` you configured while image layers
go somewhere else entirely — on Docker 28+ the containerd snapshotter keeps them
in **containerd's** root, not `data-root`.

```bash
df -h / /mnt/flock ; docker pull alpine ; df -h / /mnt/flock
```

Whichever filesystem grew is the answer. The same principle applies to Protect's
storage gate: don't trust the documented "100 GB" figure, read
`unifi-protect.service`'s `ExecStartPre` and the resulting environment:

```bash
docker exec <c> systemctl show-environment | grep UFP_RECORDING
```

---

## 11. Drive the controller from outside

An API key turns the controller into something scriptable, which pairs with §1 to
map API call → wire message.

Mint one in the console UI under Control Plane → Integrations; keys land in
`unifi-core`'s `integration_keys` table, so you can confirm registration in SQL.

```bash
curl -sk -H "X-API-Key: $KEY" https://<controller>/proxy/protect/integration/v1/meta/info
curl -sk -H "X-API-Key: $KEY" https://<controller>/proxy/protect/integration/v1/cameras
```

Available: `/v1/cameras`, `/v1/cameras/{id}/snapshot`, `/ptz/goto/{slot}`,
`/ptz/patrol/start|stop`, `/rtsps-stream`, `/v1/meta/info`.

⚠️ `ptz/goto/{slot}` returns **404 `Entity 'preset' not found`** until presets
exist — the integration API exposes no raw pan/tilt, so arbitrary movement needs
the UI or an internal endpoint. Create presets first if you want to drive PTZ this
way.

Store the key outside the project tree (`chmod 600`, root-owned) so it cannot be
committed.

---

## 12. Query the storage subsystem directly

Once the `ustate` chain is alive, it answers questions that would otherwise need
UI archaeology:

```bash
ustate dump states     # disks, flashes, raid_settings, raid_states, sdcards, spaces
ustate dump events
ustorage disk          # the real binary takes: module action [params]
uhardware block prob   # what the udev rules invoke
```

`ustate dump states` is the authoritative view of what Protect believes about
storage — bays, their `DISK_STATE_*`, and each space's `SPACE_TYPE_*`. It is also
exactly the output the wider community has been asking real-UNVR owners for.

---

## 13. Correlate an API call to the message it produces

The single fastest way to map the protocol. Drive the controller through any
client while capturing `:7442` (technique 1), and each call reveals which
`functionName` it emits.

**Why it beats reading the binary:** `ubnt_avclient` has 138 handler names and the
guides mark a dozen messages `[UNVERIFIED]` purely because nothing was known to
trigger them. A real client *is* the trigger, and it exercises paths that adoption
never touches — adoption only shows what the controller sends *unprompted*.

### Choosing a client

| Client | Language | Use it for |
|---|---|---|
| `uiprotect` | Python | **Breadth.** ~154 named `set_*` operations, so the API surface is itself the test checklist. No runtime to install where Python exists |
| `hjdhjd/unifi-protect` | TypeScript | **Streams.** The only complete livestream implementation — direct H.264/HEVC/AV1 access. Needs Node ≥22.20 |

`uiprotect` exposes named setters; hjdhjd deliberately offers a generic
`updateDevice()`, so it is the wrong shape for discovery — you would need to know
the field names already.

### The loop

```python
start = wc_l('/srv/ds/logs/ds.log') + 1     # mark the log
await cam.set_osd_name(True)                 # drive one operation
await asyncio.sleep(4)                       # let the controller act
tx, rx = frames_since(start)                 # reassemble (technique 2)
# tx = messages where from == "UniFiVideo"   (controller -> camera)
# rx = messages where from == "ubnt_avclient" (camera acks and events)
await restore()                              # put the setting back
```

**Do one operation at a time and restore after each**, or you cannot attribute a
message to a call. The 4-second wait matters: several settings are pushed
asynchronously after the HTTP call returns.

### Cautions

- **Never drive `ChangeAnalyticsSettings`.** It resets the camera. Avoid any client
  method whose name suggests analytics until you have checked where it lands.
- **Save and restore.** Read the current value before changing it; some settings
  are visible on recorded footage, and privacy masking blanks the stream.
- **Check feature flags first.** `cam.feature_flags` tells you what the model
  supports; driving `set_speaker_volume` at a camera with `has_speaker == False`
  just returns `BadRequest` and wastes a cycle.
- **Expect fan-out.** One API call can produce three messages. Do not assume a
  one-to-one mapping.
- **Expect silence.** Some settings never reach the camera at all — that is a
  result, not a failure of the capture.
