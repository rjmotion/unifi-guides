# UniFi camera — device & protocol reference

Everything established about how a Ubiquiti UniFi camera behaves as a device: its
OS, daemons, IPC bus, SSH surface, media pipeline, PTZ, detection, and — most
valuable — its failure modes.

**Subject device:** UVC G5 PTZ. Board `UVC G5 PTZ`, shortname `G5Z`,
`sysid 0xa59b` (42395). Firmware **5.3.95**, middleware **v4.70.91**. Much of
this generalises across the Gen4/Gen5 UniFi camera line; model-specific facts are
called out.

**Confidence tags.** Every claim carries one. Respect them.

| Tag | Meaning |
|---|---|
| `[MEASURED]` | Observed live on real hardware |
| `[FIRMWARE]` | Read from binaries/strings |
| `[INFERRED]` | Reasoned from the above |
| `[UNVERIFIED]` | Present in the vocabulary, never exercised |

> **Read §11 first if you are debugging.** Nearly every hour lost on this platform
> was lost to one of the failure modes documented there, and in every single case
> the fault turned out to be in the tooling rather than the camera.

---

## 1. Platform

| Property | Value | Conf |
|---|---|---|
| Kernel / arch | `Linux 4.9.84 armv7l` | [MEASURED] |
| Userland | busybox + glibc | [MEASURED] |
| Build system | OpenWrt (`openwrt-gen4s-rel`, `target-arm-openwrt-linux-gnu_glibc`) | [FIRMWARE] |
| Firmware platform tag | `sav530q`, product `uvc` | [FIRMWARE] |
| SoC family | SigmaStar ("sstar") — `ubnt_audio_agent_sstar`, encoder class `TrackEncoderVideoSAV53X` | [FIRMWARE] |
| Binary format | ELF32 ARM, PIC, Thumb-2 | [FIRMWARE] |
| SSH server | dropbear (`/etc/persistent/dropbear_ecdsa_host_key`) | [MEASURED] |
| Web server | lighttpd on :80 / :443 | [FIRMWARE] |
| Discovery | `infctld`, UDP :10001 | [FIRMWARE] |

Build paths survive in `.rodata`, giving a reliable source map:
`.../unifi-video-fw-middleware/sources/<daemon>/src/*.cpp`. [FIRMWARE]

### Busybox constraints — these shape all tooling

- **No `strings(1)`.** Static analysis requires copying binaries off-device. [MEASURED]
- **No `setsid`, no `nohup`.** A process backgrounded with `&` **dies when its SSH
  session closes.** Four independent detach strategies all failed. [MEASURED]
- **`grep -E` is unreliable** — use basic `grep` with `\|` alternation. [MEASURED]
- **`base64` / `base64 -d` are present** — this is the working bidirectional file
  transfer channel over an SSH pty. No scp needed. [MEASURED]
- Present: `ps w`, `top -b -n1`, `uptime`, `logread`, `amixer`, `arecord`,
  `timeout`, `tr`, `awk`, `sort`. [MEASURED]

### Filesystem

| Path | Nature | Conf |
|---|---|---|
| `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin` | read-only SquashFS; **identical copies** of every `ubnt_*` binary in all four | [MEASURED] |
| `/etc/persistent/` | **writable, survives reboot** — `controller.crt`, `server.pem`, `dropbear_ecdsa_host_key`, `ubnt_*.conf`, `ptz/` | [MEASURED] |
| `/etc/persistent/ptz/{dynamic,config,statistics}/` | writable — last position, presets, patrols, `overheat.log`, `totalsteps.log` | [MEASURED] |
| `/var/etc/persistent/` | alias/bind of the above | [MEASURED] |
| `/etc/avclient_state.json` | runtime adoption-state snapshot | [MEASURED] |
| `/tmp`, `/var/run` | tmpfs — `/tmp/isp_streamer_startup_state`, `/var/run/ubnt_watchdog/<daemon>/` | [MEASURED] |
| `/etc/rc.d/rc.sysinit` | boot script; **line 523** = `disable_services … ubnt_audio_events ubnt_ptz` | [MEASURED] |
| `/etc/asound.conf` | ALSA topology | [MEASURED] |

Representative persisted state [MEASURED]:

```
/etc/persistent/ubnt_avclient.conf
  { "adopted": true, "authToken": "b477cc2f…", "cfgver": 1,
    "hosts": ["wss://<CONTROLLER-IP>"], "uuid": "" }

/etc/persistent/ptz/dynamic/ubnt_lastpos.conf
  root.PTZ.PresetPos.P999.Pos=0,11012,13999,35     # (?, tilt, pan, focus)

/etc/persistent/ptz/statistics/totalsteps.log
  { "totalPanSteps": 21786, "totalTiltSteps": 10194 }
```

---

## 2. Daemon inventory

All binaries dated `May 1 2024`, owner `ui:admin`. [MEASURED]

| Binary | Size | Role | Default state |
|---|---|---|---|
| `ubnt_avclient` | 332 KB | **The hub.** The only network-facing UniFi daemon. Dials the NVR, runs adoption, relays to every sibling over IPC | running |
| `ubnt_streamer` | 216 KB | Encode + serialize; opens the outbound video push | running |
| `ubnt_ispserver` | 134 KB | Image pipeline; **owns zoom and focus** | running |
| `ubnt_ptz` | 768 KB | Gimbal (pan/tilt, presets, patrols, cruise) | **boot-disabled**, yet observed running on an adopted camera — see §11 |
| `ubnt_smart_detect` | 2.6 MB | On-device object detection (ncnn) | running |
| `ubnt_audio_agent_sstar` | 56 KB | Holds the ALSA capture device; mic pipeline owner. From inittab as `ubnt_audio_agent -p=50`, independent of adoption | running |
| `ubnt_audio_events` | 1.9 MB | Acoustic detection | **boot-disabled** |
| `ubnt_talkback` | 246 KB | Speaker / two-way audio | [UNVERIFIED] |
| `ubnt_analytics` | 447 KB | Target of `ChangeAnalyticsSettings` — **which resets the device** | [UNVERIFIED] |
| `ubnt_watchdog` | 101 KB | Per-daemon liveness; daemons checkpoint into `/var/run/ubnt_watchdog/<name>/` | running |
| `ubnt_ipc_cli` | 130 KB | **The IPC client** — see §3 | on demand |
| `ubnt_osd`, `ubnt_sounds_leds`, `ubnt_networkd`, `ubnt_cgi`, `ubnt_system_cfg` | | overlay, LED/chime, network, local API, config store | running |
| `ubnt_ctlserver`, `ubnt_reportd`, `ubnt_nvr`, `ubnt_pmask_sstar` | | roles inferred from names only | [UNVERIFIED] |

Referenced in strings but **absent on this model** (other product lines):
`ubnt_fingerprint`, `ubnt_lcm_gui`, `ubnt_mcu_agent`, `ubnt_powerd`,
`ubnt_smart_motion`, `ubnt_theta_dummy`. [FIRMWARE]

Shared libs: `libubnt_encoder.so` (612 KB — all encode/mux), `libubnt_audio_utils.so`,
`libubnt_watchdog_entity.so`, `libfdk-aac.so.2` (2.0.1), `libopus.so.0`,
`libwebsockets.so.14`. Internal libraries `ubnt_utils` (Scheduler, shellExec),
`ubnt_variant` (a JSON-ish dynamic `Variant`, plus `URI`, `CameraInfo`), and `Ipc`
(`AddHandler`, `SendRequest`, `BroadcastRequest`, `IsDestinationRunning`).
**Exactly 20 classes are shared between `ubnt_avclient` and `ubnt_ptz` — that
intersection is the IPC contract.** [FIRMWARE]

---

## 3. The IPC bus

Daemons talk over a local named-endpoint message bus. A message is a JSON object
whose only mandatory key is `functionName`; everything else is arguments.

### Invocation [MEASURED]

```sh
# fire-and-forget
ubnt_ipc_cli -T=<target> -r=0 -m='{"functionName":"...", ...}'

# request/response with timeout (ms)
ubnt_ipc_cli -T=<target> -r=1 -t=3000 -m='{"functionName":"..."}'

# payload from file — avoids ALL shell quoting of JSON, use this for anything
# non-trivial; inline quoting fails with "unknown parameter detected"
ubnt_ipc_cli -T=<target> -r=1 -t=3000 -M=/tmp/msg.json -F=-

# subscribe (register as endpoint N, print what arrives)
ubnt_ipc_cli -l=".*" -N=<unique-name>

# list reachable endpoints
ubnt_ipc_cli -L
```

### Semantics that bite

- **`-l` is a receiver, not a promiscuous tap.** It sees broadcasts plus messages
  addressed to its own registered name. A directly-addressed `A → B` message is
  **invisible**. Silence means "nothing was broadcast", *not* "nothing happened".
  [MEASURED]
- **`-r=0` success looks like:** `No response required for <Fn>. Bailing out`.
  That line means the target **parsed and dispatched** the function. Its absence
  means the name was not recognised. [MEASURED]
- **`timed out for <target>`** = the endpoint is not registered (daemon not
  running) **or** the function has no request/response handler. Identical from
  outside. [MEASURED]
- **Unknown keys are silently ignored** and still return `statusCode: 0`. A
  `statusCode: 0` reply is therefore **not** evidence your payload shape was
  right. Verify by reading back the daemon's echoed settings. [MEASURED]
- **`queueIfDestUnavailable`** queues a copy per registered-but-unavailable
  destination — the mechanism behind the starvation failure in §11. [FIRMWARE]
- Endpoints can be sub-scoped: `ubnt_ispserver:icr`. [MEASURED]
- `ubnt_avclient` registers **zero** `onRequest_*` symbols (vs four in `ubnt_ptz`);
  its routing is table-driven differently. Do not infer "can't handle X" from a
  missing symbol. [FIRMWARE]

---

## 4. SSH

### Access

SSH is **off by default** — before enablement the camera **refuses the TCP
connection** on :22 (distinguishable from auth failure). It is enabled from a
controller that has adopted the camera (`enableSsh` in the Protect app config),
not on the camera itself. User is `ubnt`; the password is the camera's **Recovery
Code**, which **rotates on re-adoption or factory reset**. [MEASURED] / the
enablement path itself [UNVERIFIED]

### Required OpenSSH options

Modern OpenSSH will not negotiate with this dropbear without: [MEASURED]

```
-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
-o PubkeyAuthentication=no -o PreferredAuthentications=password
-o HostKeyAlgorithms=+ssh-rsa
-o KexAlgorithms=+diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
```

**Trap:** `ssh-dss` is no longer valid in current OpenSSH. Including it makes
`ssh` reject the option *before connecting*, producing an error that looks
exactly like an auth failure. Use `ssh-rsa` only. [MEASURED]

Password auth non-interactively needs a pty (`sshpass`, or `pty.fork()` + select);
`ssh` will not read a password from a plain pipe. [MEASURED]

### `-tt` is mandatory for streaming reads

Without a **remote** pty, `ubnt_ipc_cli`'s stdout is a pipe and gets
**block-buffered at ~4–8 KB**. ~500-byte broadcasts then arrive in bursts of
eight-or-nothing, or vanish when the session is killed. This masquerades as
"the firmware's event broadcasts are unreliable" and is purely buffering.
[MEASURED]

### Long-running remote processes

A backgrounded process does not survive session close, and a bare
`ssh host 'ubnt_ipc_cli -l …'` **orphans** the subscriber permanently. Tie its
life to the session: [MEASURED]

```sh
ssh -tt host "sh -c 'ubnt_ipc_cli -l=\".*\" -N=<unique> 2>&1 & CP=\$!;
                     trap \"kill \$CP 2>/dev/null\" EXIT HUP INT TERM;
                     cat > /dev/null'"
```

`cat > /dev/null` blocks on session stdin; when the session drops, the trap fires.
**Use a unique `-N` name per session** — see §11.

---

## 5. Network surface

The camera **listens on very little and dials outward for everything.**

| Port | Dir | Purpose |
|---|---|---|
| 22 | in | dropbear (off until enabled) |
| 80 / 443 | in | lighttpd — the camera's local API |
| 10001/udp | in | `infctld` discovery |
| — | **out** | `wss://<nvr>:7442/camera/1.0/ws` — management/adoption |
| — | **out** | plain TCP to `avSerializer.destinations` — video push |

A full WebSocket-upgrade sweep across 18 ports × 6 paths produced **no HTTP 101**
anywhere: **the camera serves no WebSocket endpoint.** A REST sweep found **no
PTZ-capable HTTP path** — the `/cameras/{id}/ptz/position` shape belongs to the
*controller*. [MEASURED]

### The camera's local HTTPS API

| Call | Purpose |
|---|---|
| `POST /api/1.0/login` `{username,password}` | session cookie |
| `PUT /api/1.1/settings` `{controller:{addr,token}}` | **aim the camera at a controller** — this is what makes an unadopted camera dial out |
| `GET /api/1.1/settings` / `/api/1.1/status` | full readback |

Partially read-only in practice: `PUT` of `av.audio.volume=100` returns **HTTP 200
but reads back 0**; `POST` to the same path returns 500. [MEASURED]

---

## 6. Management protocol (:7442)

Only `ubnt_avclient` is network-facing, so **every controller→camera command has
exactly one entry point.** [INFERRED, strongly supported]

- **Transport:** the *camera dials the controller*, RFC 6455 upgrade on
  `GET /camera/1.0/ws`, then JSON envelopes in **binary** frames. The camera
  **does not validate the controller's TLS certificate.** [MEASURED]
- **Envelope:** camera sends `from: ubnt_avclient, to: UniFiVideo`; replies invert
  it. Replies carry `inResponseTo = <peer messageId>`; unsolicited requests use
  `inResponseTo: 0` with a fresh `messageId`. Other keys: `functionName`,
  `payload`, `responseExpected`, `timeStamp`. [MEASURED]
- **The adoption gate is replying to the camera's own hello.** The camera sticks
  in `HELLO_MSG_SENT` and retransmits `ubnt_avclient_hello` every few seconds
  until it receives a reply whose `inResponseTo` equals that hello's `messageId`.
  Sending `paramAgreement` and settings without that reply achieves nothing.
  [MEASURED]
- **The auth token is not the gate.** The camera adopts with no `?token=` in the
  URL and generates its **own** `authToken` in its `paramAgreement` reply.
  [MEASURED]
- Post-adoption: `state: AUTHENTICATION_OK`, `adopted: true`.

### The working sequence [MEASURED]

```
camera → timeSync
       ← reply; controller sends its own hello request
camera → ubnt_avclient_hello   (device announcement + capabilities)
       ← 1. reply to that hello          ◀── THE GATE
       ← 2. ubnt_avclient_paramAgreement
       ← 3. the settings suite, one at a time, ACK-GATED
camera → acks each, then emits Event* unprompted forever
```

Settings the camera cleanly acks, in order: `ResetIspSettings`,
`ChangeVideoSettings`, `ChangeDeviceSettings`, `ChangeOsdSettings`,
`NetworkStatus`, `ChangeSoundLedSettings`, `ChangeIspSettings`. See §11 for the
three that must **not** be sent.

### The hello payload — the camera announces its own limits

```json
"features": {
  "pan":  {"steps": {"min": 500,  "max": 35500}},
  "tilt": {"steps": {"min": 8000, "max": 18000}},
  "zoom": {"steps": {"min": 0,    "max": 730}},
  "mic": 1,
  "smartDetect": ["person","vehicle","animal","liveviewTracking",
                  "autoTracking","alrmSmoke","alrmCmonx","alrmBabyCry"],
  "audioCodecs": ["aac","opus"],
  "audioStyle": ["nature","noiseReduced"]
}
```

Motor `steps` are authoritative and differ per model — **read them, never
hardcode.** [MEASURED]

### Vocabulary

`ubnt_avclient` contains **138 handler names** and 472 JSON keys. [FIRMWARE]
Notable: `AbsolutePosition`, `RelativePosition`, `ContinuousMove`,
`EnablePtzControl`/`DisablePtzControl`, `Preset`, `Patrol`, `FocusTracking`,
`SpeedByZoom`, `TiltFlip`, the `Change*Settings` family (Analytics, AudioEvents,
Device, Isp, Network, Nvr, Osd, SmartDetect, SmartMotion, SoundLed, Talkback),
`Reset*` counterparts, `GetIspSettings`, `GetVideoSettings`, `GetSystemStats`,
`GetThermometerValue`, `GetIlluminance`, `UpdateFirmwareRequest`,
`ResetToDefaults`, `SetAuthToken`, `SetLEDStatus`, plus the `ubnt_avclient_*`
protocol messages (`hello`, `timeSync`, `time`, `paramAgreement`, `softRestart`,
`unmanage`).

**Events pushed unprompted** (camera→controller; the controller never polls):
`EventAnalytics`, `EventSmartDetect`, `EventSmartMotion`, `EventSmartAudio`,
`EventIspSceneStatus`, `EventIspSettingsChanged`, `EventPoorNetwork`,
`EventManagedConnStatusChanged`, `EventLensState`, `EventMotorState`,
`EventFeatureFlagsUpdated`, `EventWrongAttitude`, `EventPtzControllStatus`
*(firmware's spelling)*, `EventTalkbackState`, `EventIavState*`. [FIRMWARE]

---

## 7. Video pipeline

```
sensor → ubnt_ispserver (ISP, zoom, focus)
       → ubnt_streamer → libubnt_encoder.so
             TrackEncoderVideoSAV53X   (H.264 / H.265 / MJPEG)
             TrackEncoderAudio         (AAC via fdk-aac 2.0.1, or Opus)
             ContainerFLV              ("extendedFlv")
       → outbound TCP to avSerializer.destinations
```

Three H.264 tracks in practice: `video1` 2688×1512, `video2` 1280×720,
`video3` 640×360. [MEASURED]

### The push: outbound TCP, one port, demuxed by name

- Destinations take the form `tcp://<host>:<port>/sN`. The camera **parses the
  netloc** — the host must be the real destination address. [MEASURED]
- **Per-track destination ports are ignored.** Pointing three tracks at three
  different ports produced **zero** connections on two of them; the camera opened
  **three connections to the first port** and pushed video1/2/3 on them, then
  rewrote its own settings readback to match. **Demultiplex by FLV `streamName`,
  never by listen port.** [MEASURED]
- Each connection sends its own `onMetaData` script tag (type 18) first, carrying
  `streamName: "videoN"` — readable before any media. [MEASURED]

### `avSerializer.parameters` — the silent-drop trap

Each track's `parameters` must carry the **full five-key shape**
`{audioId, streamName, suppressAudio, suppressVideo, videoId}` (the four non-name
keys may be `null`). With only `{streamName}` the camera **silently drops the
entire `ChangeVideoSettings`** — no ack, no stream. It presents as an adoption
bug but is a payload-shape bug. **Always check for the ack.** [MEASURED]

### "extendedFlv" — the wire format

Ubiquiti's container, **not standard FLV**: [MEASURED]

- Standard 9-byte FLV header, but the **flags byte is `7`** (standard: `5`),
  then the standard 4-byte `PreviousTagSize0`.
- Standard 11-byte tag headers.
- **20 trailing bytes between tags** — a 4-byte previous-tag-size **plus a
  16-byte wall-clock trailer** — where standard FLV has 4.

Raw, `ffmpeg -f flv` fails with `Packet mismatch`. To convert: keep each tag,
re-emit a correct 4-byte previous-tag-size, drop the 16-byte trailer, force the
flags byte to `5`.

A separate MJPEG track is pushed on its own connection; frames are recoverable by
scanning SOI/EOI (`FFD8`…`FFD9`). [MEASURED]

### Exposure

`ChangeIspSettings` carries ~60 imaging fields. Measured `aeTargetPercent` vs
mean frame luma Y: [MEASURED]

| `aeTargetPercent` | 20 | 35 | 50 | 65 | 80 | 95 |
|---|---|---|---|---|---|---|
| mean luma Y | ~101 | ~108 | ~121 | ~135 | ~148 | ~162 |

Monotonic and real, but **linear in luma, not logarithmic**, and compressed at the
dark end (20→35 moves only ~7 luma). **The sensor does not deliver photographic
"stops".** `enableLogo: 0` in `ChangeOsdSettings` removes the watermark;
`enableOverlay: 1` must stay on for anything to render.

---

## 8. Audio

`feature_mic = 1`; the pipeline **runs** — `ubnt_audio_agent_sstar` holds the ALSA
capture (state `RUNNING`), `ubnt_streamer` is up. [MEASURED]

### ALSA topology (`/etc/asound.conf`)

| PCM / control | Role |
|---|---|
| `pcm.ubnt_capture` | **the real mic capture device** |
| `pcm.ubnt_cvolume` → control **`UBNT_CVOLUME`** | softvol on the capture path |
| `ubnt_snoop` | dsnoop shared-capture layer feeding `ubnt_cvolume` |
| `ubnt_capture_talkback` | talkback PCM |
| `ubnt_capture_loopback` | referenced by `ubnt_audio_events` but **does not exist** in this firmware |

`UBNT_CVOLUME` is directly settable: `amixer -D default sset UBNT_CVOLUME 100%`
moved it 0 → 255. [MEASURED]

### The two volumes — the crucial distinction

| Value | Controls | Settable? |
|---|---|---|
| **`UBNT_CVOLUME`** (ALSA softvol) | the *level* of captured audio; 0 = silence | **YES**, via `amixer` |
| **`av.audio.volume`** | whether an audio track is **muxed at all** (read by `TrackEncoderAudio::Muted()`) | **NO** |

### Measured outcome: a third-party controller cannot get in-band audio

Every lever, tested end to end: [MEASURED]

| Lever | Result |
|---|---|
| `suppressAudio:false` + `audioId:1000` per track | echoed back verbatim — **0 audio tags** |
| **`hasAudio:true`** on the serializer params | **0 audio tags** (the firmware analysis's predicted fix failed) |
| per-track `audio{enabled:true,…}` sub-object | camera **drops** it — audio config is device-level only |
| `av.audio.volume:100` in `ChangeVideoSettings` | camera **force-resets to 0** in the same reply, every time |
| `PUT /api/1.1/settings` volume=100 | HTTP 200, stays 0; POST → 500 |
| `amixer sset UBNT_CVOLUME 100%` | **works** — un-mutes *capture*, still 0 audio tags |

Full 2 MB captures: ~100–118 video tags, **0 audio tags**, every run. `ffmpeg`
reports `video:NNNkiB audio:0kiB` — a declared track with zero frames, since the
camera's own `onMetaData` advertises `hasAudio:true, 16000 Hz, AAC`.

`av.audio.volume` is **not** in `systemcfg`, `/etc/persistent`, or `/var/etc`. It
appears to be controller-managed runtime state a third party cannot reproduce.

**The only remaining route to mic audio** is to raise `UBNT_CVOLUME` and read the
`ubnt_capture` ALSA device directly over SSH, bypassing the camera's muxer.
[INFERRED, not built]

Talkback (`ChangeTalkbackSettings`, `InitTalkbackPCM`, `withTalkback`,
`feature_aec`) is present and entirely [UNVERIFIED].

---

## 9. PTZ

**Pan/tilt belong to `ubnt_ptz`. Zoom/focus belong to `ubnt_ispserver`.** [MEASURED]

Underneath, the motor driver speaks **VISCA** (`visca_G5PTZFLEX.h`,
`CAM_PanTiltPos_Inq`, …) with an MCU behind it. `ubnt_ptz` is a translation
layer: IPC in → VISCA out. [FIRMWARE]

### The one command proven to move the gimbal [MEASURED]

```sh
ubnt_ipc_cli -T=ubnt_ptz -r=0 -m='{"functionName":"AbsolutePosition",
   "panPos":<int>,"tiltPos":<int>,"panSpeed":<int>,"tiltSpeed":<int>}'
```

**Omitting `panSpeed`/`tiltSpeed` makes the whole move a no-op.** Zoom is a
different message to a different daemon:
`ubnt_ipc_cli -T=ubnt_ispserver -m='{"functionName":"ChangeIspSettings","zoomPosition":<int>}'`

Positions are **raw integer motor coordinates**, never normalised. **Tilt travel
is asymmetric** (≈ −10°…+90°), so the numeric midpoint is *not* level.

### Speed is a wide linear scale [MEASURED]

| `panSpeed` | slew time over 20 000 units |
|---|---|
| 5 / 10 / 24 | > 80 s |
| 100 | ~65 s |
| 255 | ~35–40 s |
| **500** | **≤ 20 s — saturation** (matches the boot self-test speed) |
| 2000 / 10000 | identical to 500 |
| omitted | **NO-OP** |

### Position feedback is broadcast-only — never queryable

Position lives in **shared memory** (`PTZSharedMemory::getPtzfPos`), not on the
bus. There is **no request/response handler** for it, which is why
`GetCurrentPosition` and friends all time out — they are internal C++ symbols,
not IPC endpoints. [FIRMWARE] + [MEASURED]

Instead `MotorStateMonitor` **broadcasts** `EventMotorState`:

```json
{ "state": { "position": {"pan":35008,"tilt":10054,"zoom":0,"focus":41},
             "activity": 12, "wallClockMs": 946707833694 } }
```

`activity` is **non-zero while moving, 0 when settled** — a real motion-complete
signal. `wallClockMs` uses the camera's wrong clock (§11).

> **The operational consequence, which has cost multiple debugging sessions:**
> a still head broadcasts **nothing**. You cannot move and then read — by the time
> you read, the motion is over. The order must be **start the subscriber → confirm
> it is streaming → fire the move → capture broadcasts during the motion window.**
> Poll-after-move reads as "the gimbal didn't move" on a perfectly healthy gimbal.

### Negative results [MEASURED]

`ContinuousMove`, `PanTiltReset`, `PresetGo` as guessed were accepted-looking but
**no-ops**. `GetCurrentPosition` times out. `panTiltStop` / `zoomStop` **are**
acknowledged and do interrupt an in-flight move.

### `PtzWebsockHandler` — the biggest open question

`ubnt_ptz` contains a complete WebSocket **client** that dials out to a `wss://`
URI it is given, retries with backoff, presents `Camera-MAC`/`hwaddr` headers plus
`/etc/persistent/server.pem` + `controller.crt`, and speaks the same
`messageId`/`inResponseTo`/`UniFiVideo` envelope wrapped in
`fromControllerRequest` / `toControllerResponse`. [FIRMWARE]

**What supplies that URI is unknown.** Ruled out [MEASURED]: PTZ over the
management WebSocket (identical payload moved the head 1310 units over IPC, **0**
over :7442); a camera-hosted PTZ port (it listens on nothing suitable);
`EnablePtzControl{uri}` over :7442 (no ack, no dial — and its handler lives *in*
the WebSocket file, so it arrives *on* the socket and cannot be what opens it);
the PTZ config files. `URI` is one of the 20 classes shared with `ubnt_avclient`,
so it **can** cross the IPC boundary as a typed value.

Best remaining lead: **observe a real UniFi Protect controller adopt this camera
and capture what it sends.**

---

## 10. Detection

### Object detection

The daemon **runs by default** with a fully populated config — **except
`enableSmartDetect: []`**. The detector runs but no class is active, so nothing
fires. The on-switch is **`ChangeSmartDetectSettings`** (fields
`enableSmartDetect`, zone `objectTypes`, `sensitivity`); it applies live, is
acked, and leaves the connection **stable**. [MEASURED]

Once enabled the camera pushes `EventSmartDetect` (with `objectTypes` and
`edgeType` = enter/leave) and `EventAnalytics` for plain motion (`edgeType` =
start/stop), unprompted.

- **Detector input is a 640×360 preview.** Distant subjects fall below the model's
  floor — a viewpoint limit, not a fault. [MEASURED]
- Enabling it costs **~46% CPU**. [MEASURED]
- It logs to `/var/log/smart_detections.log` and checkpoints into
  `/var/run/ubnt_watchdog/ubnt_smart_detect/` every ~27 s. **A stale checkpoint
  plus flat CPU time is the signature of a wedged detector.** [MEASURED]

### Acoustic detection

`ubnt_audio_events` is **boot-disabled**, therefore not a registered endpoint —
every message returns `timed out`. Started by hand it **works on this hardware**
(opens ALSA `ubnt_capture`, loads ncnn models, initialises `babycry_event_detect`);
the per-platform gates are not hit on this model. [MEASURED]

**`ChangeAudioEventsSettings` — recovered schema.** A **flat** settings object;
no wrapper, no `zones`, no list. Each class is its own key: [MEASURED]

```jsonc
{ "enableAlrmSmoke":0|1, "enableAlrmCmonx":0|1, "enableAlrmSiren":0|1,
  "enableAlrmSpeak":0|1, "enableAlrmBabyCry":0|1, "enableAlrmBurglar":0|1,
  "enableAlrmCarHorn":0|1, "enableAlrmBark":0|1, "enableAlrmGlassBreak":0|1,
  "enableLoudNoise":0|1, "enableSoundLoss":0|1,
  "thresholdEventSiren":80, "thresholdEventBurglar":50, "thresholdEventCarHorn":90,
  "thresholdEventSpeak":90, "thresholdEventBabyCry":85, "thresholdEventBark":90,
  "thresholdEventGlassBreak":98, "smokeThreshold":-20, "cmonxThreshold":-20,
  "eventThreshold":50, "loudThreshold":80, "quietThreshold":0,
  "recordEvent<Class>":0, "linger<Class>StartSec":0, "linger<Class>StopSec":10..24,
  "levelUpperBound":-10, "levelLowerBound":-50, "leveldBShifter":-10,
  "sendPulse":1, "pulsePeriodSec":5 }
```

**Value type matters: integers `0|1`, not booleans.** `{"enableAlrmSiren": true}`
is silently ignored (echoes back `0`); `{"enableAlrmSmoke": 1}` applies and
persists. [MEASURED]

**Settings input vocabulary ≠ event output vocabulary.** Events return as
`EventSmartAudio` using the **bare** names — `alrmSmoke`, `alrmCmonx`, `alrmSiren`,
`alrmSpeak`, `alrmBabyCry`, `alrmBurglar`, `alrmCarHorn`, `alrmBark`,
`alrmGlassBreak`, `loudNoise`, `soundLoss` — plus `eventType`, `eventId`,
`duration`, `levels`, `leveldB`, `enter`, `leave`. [FIRMWARE]

---

## 11. Failure modes — read this first

### Messages that reset the device

| Message | Behaviour |
|---|---|
| **`ChangeAnalyticsSettings`** | **Resets the camera** — connection drops the instant it arrives, empty payload *or* full. Measured twice, producing a 14-handshake storm. [MEASURED] |
| **`ChangeAudioEventsSettings`** | Tears down the control channel → ~7 s reset loop, **because `ubnt_audio_events` is boot-disabled**. Sent directly to a running daemon, every shape returns `statusCode 0` — the fault is in `ubnt_avclient`'s forwarding path, not the schema. [MEASURED] |
| `UpdateUsernamePassword` | Never acked; contributes to the trailer reset below. [MEASURED] |
| `ChangeSmartDetectSettings` | Acked, applied live, **stable**. Safe. [MEASURED] |

### The ~6-second reset loop

Blasting the settings suite as one unsolicited burst makes the camera reset :7442
every ~5–6 s forever (21 000+ handshakes observed). Root cause: it **cleanly acks
the first seven**, then **never acks** `ChangeAnalyticsSettings`,
`ChangeSmartDetectSettings`, `UpdateUsernamePassword`, resetting **~2 s** after
receiving them — far too fast for a keepalive timeout. It is actively rejecting.

**Disproven, do not revisit:** it is *not* a malformed `ChangeVideoSettings` (that
is acked), and *not* a missing keepalive (pinging every 2 s changed nothing).

**Mitigation: ack-gate the suite** — send N, wait for a receive with
`inResponseTo == N`, then send N+1 — and drop the trailers. [MEASURED]

### Adoption UUID persistence — the silent wedge

The camera **persists the controller UUID it adopted under** and enforces a match
(`UUID mismatch: has %s, received %s`). On mismatch it connects, sends hello, and
then **acks nothing** (zero `inResponseTo` across 334 000 frames), emits **zero**
events, **never pushes video**, and resets every ~6 s. **The wedge survives a
reboot** — that persistence is the tell that the cause is stored state. Presenting
the persisted UUID restores everything instantly. [MEASURED]

**Rule: the controller UUID must never drift between runs.**

### Leaked IPC subscribers starve the bus

**The single most misleading failure mode on this device.** [MEASURED]

Every SSH session that starts `ubnt_ipc_cli -l -N=<name>` and dies without killing
the remote process leaves the subscriber **registered forever**. `ubnt_ptz` then
unicasts every broadcast to each registered endpoint with
`queueIfDestUnavailable: true`, so dead subscribers (31 at peak) drown the
broadcaster and **live** listeners starve. Each session makes the next worse.
Observed load average ~16 sustained, ~78 with detection also enabled. A **fixed**
subscriber name additionally collides with its own leaked registration.

Mitigations, both required: the trap-wrapper from §4, **and a unique subscriber
name per session.**

> **The general lesson, which held every single time:** when this camera appears
> broken, the fault has been in the tooling. Retracted misdiagnoses: "the gimbal
> is hardware-faulted" (was poll-after-move); "position feedback is flaky on this
> firmware" (was leaked subscribers + block buffering); "the WebSocket definitively
> cannot carry PTZ" (was a malformed empty payload).

### Wedged subsystems

- **`ubnt_ptz` position reporting can wedge** — motors fine, commands recognised,
  but `EventMotorState` frozen for ~40 min. Recovery: reboot. [MEASURED]
- **`ubnt_smart_detect` can stall at init** — log frozen at its init timestamp
  while video keeps flowing, flat CPU, stale watchdog checkpoint. [MEASURED]
- **The `ptResult` calibration readback is NOT a "gimbal dead" signal** — it read
  non-zero while the head moved perfectly. Never diagnose motion from a readback;
  move and observe. [MEASURED]

### Reboot

`reboot` over SSH works; :22 drops within seconds and returns in **60–120 s**, and
the camera re-adopts automatically. Allow ~30 s after :22 returns. [MEASURED]

### The clock — no NTP

The clock sits at the **2000-01-01 power-on epoch plus uptime**. The camera
**ignores the `timeDelta` in a `timeSync` reply** (72 000+ sends, never moved). No
NTP daemon runs — only an uninvoked `/sbin/ntpclient`. A burst of ~10 rapid
`timeSync` exchanges per connection is **normal** sampling, not a fault.
Consequence: the burned-in OSD date is wrong and `wallClockMs` is meaningless as
absolute time. **Video is otherwise unaffected.** Untested fixes: serve NTP, or
`ChangeNvrSettings.ntpClientHost`. [MEASURED] / fixes [UNVERIFIED]

---

## 12. Firmware image

Metadata from `https://fw-update.ui.com/api/firmware-latest`: product `uvc`,
platform `sav530q`, version `v5.3.95`, 58 098 688 bytes. [MEASURED]

### Format — XOR-obfuscated, not encrypted

- **64-byte ASCII-hex header** (per-image key/IV), then the body.
- Body obfuscated with a **period-16 repeating XOR keystream**. Evidence it is not
  AES: entropy 7.7–7.99 rather than flat 8.0; a 16-byte block repeats ~235 000
  times; a 1-byte plaintext change flips exactly 1 ciphertext byte; autocorrelation
  spikes at lag 16.
- **Recovered key `1a0713191a071319d7d5d7da1a071319`**, applied from body offset
  `0x40`. Recovered by autocorrelation plus known-plaintext against `0xFF` padding.
- `0x000000–0x300000` decrypts cleanly (init scripts, **inittab**, lighttpd conf,
  passwd hashes). `0x600000–0x3600000` is **compressed SquashFS** — the `hsqs`
  superblock never appears in plaintext, so the ELFs inside need
  `unsquashfs`/`sasquatch`/`binwalk`. [MEASURED]

### The reliable route to binaries

**Pull them off the live camera, base64-encoded** — no scp, no extra tooling:

```sh
base64 /usr/sbin/ubnt_ptz        # filter output to ^[A-Za-z0-9+/=]+$, decode
```

Verified for `ubnt_avclient`, `ubnt_ptz`, `ubnt_streamer`, `ubnt_smart_detect`,
`ubnt_audio_events`, `libubnt_encoder.so`. Analyse off-device (no `strings(1)` on
the camera). Demangling Itanium C++ names plus the surviving `__FILE__` paths
yields a reliable class/source map. [MEASURED]

---

## 13. Open questions

1. **What supplies the `PtzWebsockHandler` URI?** The largest gap — a complete
   outbound PTZ WebSocket client that never dials. Best lead: watch a real
   controller adopt this camera.
2. **`av.audio.volume`** — where does it live, what sets it? Not in `systemcfg`,
   `/etc/persistent`, or `/var/etc`.
3. **`ubnt_ptz` boot state.** `rc.sysinit:523` disables it, yet it runs and
   responds on an adopted camera. What re-enables it — `ubnt_avclient`'s "enable
   related processes" path? The same mechanism would presumably start
   `ubnt_audio_events`.
4. **Native PTZ preset/patrol/cruise/autopan** — the whole `Exec*` family is
   present and persisted, every one untested. Enumeration may require reading the
   config files, since `PresetList` needs a reply the daemon may not serve.
5. **`ContinuousMove` / `RelativePosition` argument shapes** — never determined;
   guessed shapes no-op'd.
6. **Talkback** — completely untested.
7. **`ChangeNvrSettings`** — neighbouring keys include `rtspEnabled`, `rtspPort`,
   `rtspUsername`, `rtspPassword`, `ntpClientHost`, `recordFulltime`. **The
   presence of `rtspEnabled`/`rtspPort` suggests the camera may be able to serve
   RTSP directly.** Never tested — potentially the highest-value unexplored lead.
8. **Untested messages:** `ChangeEventSettings`, `ChangeAvclientEventSettings`,
   `ChangeNetworkSettings`, `ChangeSmartMotionSettings`, `GetRequest` (reportedly a
   snapshot-upload trigger), `ubnt_avclient_softRestart`, `ubnt_avclient_unmanage`,
   `ResetToDefaults`, `ResetVideoDestinations`, `SetLEDStatus`, `GetSystemStats`.
9. **Why `hasAudio:true` did not unblock the audio mux** despite the firmware gate
   chain saying it should — the discrepancy is unexplained.
10. **The `activity` field's full encoding** in `EventMotorState` — `0` = settled,
    `12` = moving are established; `8` appears in other states. Treat as a flag
    word, not a magnitude.
11. **Why fresh IPC subscribers sometimes receive only ~one broadcast per session**
    even on a purged bus — observed, unexplained, do not build on it.
