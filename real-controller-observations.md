# Observing a real controller — measured findings

What a genuine UniFi Protect controller does to a UVC G5 PTZ, captured by running
Protect off-hardware (see [`../README.md`](../README.md) part one) and reading its
own logs while it adopted the camera.

This is the experiment the other guides kept asking for. Both
[`unifi-camera-reference.md`](unifi-camera-reference.md) §9 and §13 end their
hardest open questions with the same line:

> **Best remaining lead: observe a real UniFi Protect controller adopt this camera
> and capture what it sends.**

That has now been done, and it **answers the largest open question in the project**
and **overturns several documented conclusions**, including one whole section
heading.

---

## Provenance and confidence

| | |
|---|---|
| Controller | UniFi Protect **7.1.77** + unifi-core 5.1.117, containerised on a Raspberry Pi CM4 (arm64), no Ubiquiti hardware |
| Camera | **UVC G5 PTZ**, MAC `<CAMERA-MAC>`, at <CAMERA-IP> — the guides' device of record |
| Controller address | <CONTROLLER-IP> |
| Protect device id | `<CAMERA-ID>` |
| Date | 2026-07-25 |
| Source | `cameras.avclient.log`, `cameras.log`, and — for §9 — **plaintext WebSocket frames** from `ds`'s own `trace.log_websocket_payload`, plus a `:7550` packet capture |

### Placeholders — how to get your own values

Device identifiers are deliberately not published; they identify the machine a
measurement ran on, not the finding. Substitute your own:

| Placeholder | What it is | How to obtain it locally |
|---|---|---|
| `<CONTROLLER-IP>` | The address your controller listens on — the one cameras dial | `ip -4 addr show scope global` on the controller host. Must be the address the camera can reach; the camera parses the netloc and connects to it literally |
| `<CAMERA-IP>` | The camera's own address | Your DHCP server's lease table, or the Protect UI's device page. The camera only listens on 22/80/443 and udp/10001 (§5 of the camera reference), so a sweep for tcp/443 finds it |
| `<CAMERA-MAC>` | The camera's hardware address | Printed on the device label; also sent by the camera itself in the `camera-mac` WebSocket upgrade header and as `hwaddr` in its `ubnt_avclient_hello` (§3 of the protocol reference) |
| `<CAMERA-ID>` | Protect's *internal* id for the camera — not a hardware value | `curl -sk -H "X-API-Key: $KEY" https://<CONTROLLER-IP>/proxy/protect/integration/v1/cameras`. Mint the key in the console UI under Control Plane → Integrations |
| `<AUTH-TOKEN>` | Per-adoption token the camera generates for itself | Read `/etc/persistent/ubnt_avclient.conf` on the camera over SSH. **Rotates on re-adoption and factory reset**, so it is worthless to anyone else — and per §3, it is not what gates adoption anyway |

**A note on evidence class.** These are **controller-side log statements** of the
form `SENT <functionName> with payload <json>` — Protect reporting what it sent,
not bytes captured off the wire. That is strong evidence of the message vocabulary
and payload shapes, but it is one step removed from the transport. Anything
depending on exact framing, ordering under load, or ack behaviour should be
confirmed with a packet capture before being relied on.

Tagged `[MEASURED-LOG]` in §§1–8 to keep this distinct from the `[MEASURED]`
(observed on the wire / on hardware) used elsewhere in `guides/`.

**[§9](#9-plaintext-capture--measured) is different.** It is built from 263
plaintext JSON envelopes and a packet capture, so it is tagged `[MEASURED]` — and
it **corrects two claims made in §3**. Where the two sections disagree, §9 wins.

---

## 1. The `PtzWebsockHandler` URI — **SOLVED**

`unifi-camera-reference.md` §13 lists this first:

> **What supplies the `PtzWebsockHandler` URI?** The largest gap — a complete
> outbound PTZ WebSocket client that never dials.

**The answer:**

```json
EnablePtzControl  {"uri": "wss://<CONTROLLER-IP>/camera/1.0/ws"}
```

Sent by the controller over the **management channel on :7442**, roughly 90
seconds into adoption. `DisablePtzControl` is its counterpart and carries **no
payload at all**. [MEASURED-LOG]

Two things about this matter:

1. **The URI is the *same* management WebSocket path** — `/camera/1.0/ws` on the
   controller — not a separate PTZ endpoint on another port. The PTZ client dials
   back to the same place the management channel already lives.
2. **The guides explicitly ruled this mechanism out**, and the reasoning was
   wrong. `unifi-camera-reference.md` §9 records under "Ruled out [MEASURED]":

   > `EnablePtzControl{uri}` over :7442 (no ack, no dial — and its handler lives
   > *in* the WebSocket file, so it arrives *on* the socket and cannot be what
   > opens it)

   The real controller sends exactly this, over exactly that socket. The
   "arrives on the socket so cannot open it" argument does not hold — the
   handler receiving the message and the client it then dials are separate
   concerns. The earlier negative was most likely a payload-shape or
   sequencing problem, not a structural impossibility.

**For `cuckoo`:** send `EnablePtzControl` with the controller's own
`wss://<addr>/camera/1.0/ws` after adoption completes.

---

## 2. PTZ *is* on this protocol — §8 of the protocol reference is wrong

[`protocol-reference.md`](protocol-reference.md) §8 is titled **"PTZ is not on
this protocol"**, on the strength of an `AbsolutePosition` payload moving the head
1310 units over SSH/IPC and 0 over :7442.

The real controller drives PTZ over :7442 throughout. Observed [MEASURED-LOG]:

| Message | Payload | Count |
|---|---|---|
| `GetCurrentPosition` | `{"inDegree":true,"inSteps":true}` | 3 |
| `EnablePtzControl` | `{"uri":"wss://<CONTROLLER-IP>/camera/1.0/ws"}` | 2 |
| `DisablePtzControl` | *(none)* | 2 |
| `Preset` | `{"trackTimeoutSec":20,"backToPresetPosition":true}` | 1 |
| `ChangePTZAutoTrackSettings` | `{}` | 2 |
| `EventMotorState` | *(camera → controller)* | **32** |

**`GetCurrentPosition` deserves particular attention.**
`unifi-camera-reference.md` §9 states:

> There is **no request/response handler** for it, which is why
> `GetCurrentPosition` and friends all time out — they are internal C++ symbols,
> not IPC endpoints.

The real controller calls it three times during adoption with
`{"inDegree":true,"inSteps":true}`. **The payload shape is the difference** — the
earlier attempts appear to have sent it bare. Note it requests *both* unit
systems explicitly; a request specifying neither may well be rejected.

The 32 `EventMotorState` messages confirm the guides' broadcast-only model for
position feedback, and confirm it flows over :7442 rather than only over IPC.

---

## 3. Streaming — four corrections

From `cameras.log`, the ingest configuration pushed to each channel:

```json
{"video":{"video3":{"avSerializer":{
  "type":"extendedFlv",
  "parameters":{"streamName":"FHkKR8pYKdLq1kk6","withOpus":true,"opusSampleRate":24000},
  "destinations":["tcp://<CONTROLLER-IP>:7550?retryInterval=1&connectTimeout=5"]},
  "type":"h265"}}}
```

### 3.1 Single push port — confirmed **for this model**

All three channels were given **`tcp://<CONTROLLER-IP>:7550`**, and three TCP
connections from the camera to :7550 were established. [MEASURED-LOG] + observed
via `ss`

This confirms the single-port behaviour recorded in `protocol-reference.md` for
the **G5 PTZ**. It does **not** settle §9 open question 5, and the earlier framing
of this as a disagreement to be won was wrong.

`scrypted-unifi-direct` reports per-track ports **17550–17552 working** on a G5
Turret Ultra (fw 5.3.90). A working implementation is evidence, not a mistake —
so the honest reading is that **this is model- or firmware-dependent**, and both
observations stand on their own hardware.

**What that means for a receiver:** don't hardcode either. Set the destinations
you want, then demultiplex by the `streamName` you assigned, and be prepared for
the camera to collapse every track onto the first port regardless of what you
asked for. That works under both behaviours; keying on port only works under one.

### 3.2 The codec is **H.265**, not H.264

Tracks are configured `"type":"h265"`. The guides describe the push as H.264
throughout, including the AVC-sequence-header logic for finding a decodable
prologue. **A receiver written to the existing guide will not decode this
stream.** [MEASURED-LOG]

### 3.3 `streamName` is a random per-session token

Observed: `FHkKR8pYKdLq1kk6`, `OQ5tZVhptKAFSkWG`, `g8EwaDub9uObvSjQ` — 16
characters, mixed case, one per channel. **Not** the literal `"video1"` /
`"video2"` / `"video3"` recorded in the guides. Demuxing by name still works, but
only because the controller chose the names; a receiver must map the names it
assigned rather than expect fixed ones. [MEASURED-LOG]

### 3.4 Audio is **Opus** — and this is probably the missing lever

`unifi-camera-reference.md` §8 concludes, after an exhaustive table of failures,
that **"a third-party controller cannot get in-band audio"** — `suppressAudio`,
`audioId`, `hasAudio`, per-track `audio{}`, `av.audio.volume`, `amixer`, all
tried, all yielding 0 audio tags against an advertised AAC track.

The real controller never asks for AAC. It sends:

```json
"parameters": {"streamName":"…", "withOpus":true, "opusSampleRate":24000}
```

**`withOpus` does not appear anywhere in the existing guides.** The camera's own
hello advertises `"audioCodecs":["aac","opus"]`. [MEASURED-LOG]

> ### ⚠️ Superseded — see §9
>
> A packet capture of `:7550` has since **overturned §8's conclusion outright**:
> audio tags *do* flow, ~950–1100 per stream. But **not as Opus** — the wire
> carries **AAC-LC, 16 kHz, mono**, despite `withOpus:true` being requested.
>
> So "withOpus is the missing lever" was **wrong**. The likely lever is the
> *shape of the audio object*, not the codec — see §9.3. [MEASURED]

### 3.5 Destinations use query parameters, not a path suffix

`tcp://<host>:7550?retryInterval=1&connectTimeout=5` — the guides record
`tcp://<host>:<port>/sN` and note the `/sN` suffix is cosmetic. The real
controller uses neither a path nor `/sN`, but two tuning parameters. [MEASURED-LOG]

### 3.6 `avSerializer.parameters` — the "mandatory five-key shape" is not mandatory

The guides state this emphatically, twice
([`protocol-reference.md`](protocol-reference.md) §4,
[`unifi-camera-reference.md`](unifi-camera-reference.md) §7):

> `avSerializer.parameters` **must carry the full five-key shape** — `audioId`,
> `streamName`, `suppressAudio`, `suppressVideo`, `videoId` … With only
> `{streamName}` the camera **silently drops the entire message**.

The real controller sends **three keys**: `streamName`, `withOpus`,
`opusSampleRate`. None of `audioId`, `suppressAudio`, `suppressVideo` or
`videoId` appear, and the stream works. [MEASURED-LOG]

Either the rule was always wrong, or it is firmware/version-dependent. Given how
much debugging time the guides attribute to it, **this is worth re-testing
deliberately.**

---

## 4. The adoption handshake — confirmed, with the gate exactly as described

Ordered from `cameras.avclient.log`: [MEASURED-LOG]

```
  camera                                   controller
  ──────                                   ──────────
                                     addConnection, mac: <CAMERA-MAC>
  ubnt_avclient_hello   (id 81068042) ──►
                                      ◄──  ubnt_avclient_hello  (id 10010)   ◀══ THE GATE
  EventIspSceneStatus   (id 81068043) ──►
                                      ◄──  ubnt_avclient_paramAgreement (id 10011)
                                           "authenticated, uptime 58"
  EventPoorNetwork      (id 81068045) ──►
                                      ◄──  GetCurrentPosition {"inDegree":true,"inSteps":true}
                                      ◄──  the settings suite …
```

**The guides' central claim about the gate is correct**: the controller replies to
the camera's own hello, and everything follows from that. Controller message ids
start at a low counter (10010, 10011 …) while the camera's are large
(81068042 …) — consistent with the guides' "monotonic per sender, any integer".

Note **`authenticated` is logged immediately after `paramAgreement`**, before the
settings suite — so authentication completes at paramAgreement, not at the end of
settings.

### Settings suite, in the order actually sent

```
 1 GetCurrentPosition          9 ChangeBrightnessSettings
 2 ChangeVideoSettings        10 ChangeSoundLedSettings
 3 StopService                11 ChangeTalkbackSettings
 4 ChangeIspSettings          12 ChangeVideoSettings
 5 ChangePTZAutoTrackSettings 13 AudioAgentChangeTuning
 6 ChangeDeviceSettings       14 ChangeIspSettings
 7 ChangeDeviceSettings       15 ChangeVideoSettings
 8 ChangeOsdSettings          16 ChangeSmartMotionSettings
                              17 DisableLogging
                              18 ChangeDeviceSettings
```

`ChangeVideoSettings` is sent **12 times** in total and `ChangeIspSettings` 4
times — the controller re-pushes video configuration repeatedly rather than
once, which fits the "camera echoes it back, that echo is authoritative" model.

---

## 5. Vocabulary the guides do not document

All observed being sent by the real controller. [MEASURED-LOG]

| Message | Payload | Note |
|---|---|---|
| **`StopService`** | `{"service":"ssh"}` | **The controller turns SSH *off* on adoption.** Directly relevant to `unifi-camera-reference.md` §4, which records SSH as off by default and enabled from the controller — this is the mechanism, in the disable direction |
| `DisableLogging` | `{}` | |
| `ChangePTZAutoTrackSettings` | `{}` | PTZ auto-tracking |
| `ChangeBrightnessSettings` | — | separate from `ChangeIspSettings` |
| `ChangeClarityZones` | — | |
| `ChangeInterfaceSettings` | — | |
| `AudioAgentChangeTuning` | — | targets the audio agent |
| `SmartMotionTest` | — | |
| `UpdateFaceDBRequest` | — | face recognition |
| `SendWeatherUpdate` | — | controller pushes weather to the camera |

### Hazard list needs revising

`protocol-reference.md` §6 marks `ChangeAudioEventsSettings` as hazardous —
"tears down the control channel → ~7 s reset loop". **The real controller sends
it during adoption** without incident. The guides' own root-cause note explains
why: the fault was that `ubnt_audio_events` was boot-disabled on the test rig, so
the message had no recipient. Against a properly-brought-up camera it is safe.

`UpdateUsernamePassword` is also sent, which the guides list among the
"never acked, contributes to the reset" trailer messages.

---

## 6. What this changes for `cuckoo`

1. **PTZ is achievable.** Send `EnablePtzControl{uri}` pointing at your own
   `wss://<addr>/camera/1.0/ws`. This is the single highest-value change.
2. **Expect H.265.** The existing H.264/AVC prologue logic will not work as
   written.
3. **Ask for Opus** via `withOpus`/`opusSampleRate` rather than fighting AAC.
4. **Stop sending the five-key `parameters` shape** as if mandatory; three keys
   are sufficient for the real controller.
5. **Use `GetCurrentPosition` with `{"inDegree":true,"inSteps":true}`** rather
   than treating position as broadcast-only.
6. **One push port.** The existing single-port/`streamName` design is correct.

---

## 7. Open, and now testable

The controller is live and driveable via the integration API
(`X-API-Key`, `https://<host>/proxy/protect/integration/v1/`), with
`/v1/cameras/{id}/ptz/goto/{slot}`, `/ptz/patrol/start|stop`, `/snapshot` and
`/rtsps-stream`. That makes the following straightforward experiments:

1. **Packet-capture :7442 during a PTZ command** to get wire-level framing and
   ack behaviour, upgrading everything here from `[MEASURED-LOG]` to `[MEASURED]`.
2. **Does `withOpus` actually yield audio tags?** Capture :7550 and count audio
   tags. This is the direct test of §3.4.
3. **What does `Preset` do**, and how do preset slots map to
   `/v1/cameras/{id}/ptz/goto/{slot}`?
4. **Re-test the five-key `parameters` rule** deliberately, to determine whether
   it was ever true.
5. **`AbsolutePosition` over :7442** — given `GetCurrentPosition` works with the
   right payload, re-test absolute moves rather than treating §8 as settled.
6. **What re-enables SSH**, given `StopService{"service":"ssh"}` disables it.

---

## 9. Plaintext capture — `[MEASURED]`

Everything above is `[MEASURED-LOG]`. This section is **wire-verified**, and it
overturns several conclusions — including two of my own from §3.

### 9.1 Method: `ds` will log its own plaintext. No MITM, no debugger.

`:7442` is TLS, so a packet capture yields ciphertext. `:7550` is plain TCP and
fully readable. For the management channel, the useful discovery is that the
daemon serving `:7442` — **`ds`**, a Rust binary using `tokio-tungstenite` — has
payload tracing built in and switched off by default:

```jsonc
// /usr/share/ds/ds.json
"trace": {
  "log_websocket_payload": false,   // -> true: hex+ASCII frame dumps in /srv/ds/logs/ds.log
  "dump_websocket_payload": false,  // -> true: JSON envelopes in /srv/ds/logs/payloads.jsonl
  "auto_dump_websocket_threshold": 50
}
```

Set both to `true`, `systemctl restart ds`, and the complete JSON envelopes appear
in the clear, both directions. **263 envelopes** were recovered this way by
reassembling the hex dumps.

Worth knowing why the alternatives fail: `ds` links **no libssl**, so there are no
`SSL_read`/`SSL_write` symbols for `LD_PRELOAD` or a debugger to hook. A TLS MITM
would work — the camera doesn't validate the controller's certificate — but is
unnecessary given the above.

`dump_websocket_payload` is event-gated and captured only `EventSmartDetect`;
`log_websocket_payload` is the one that yields everything.

### 9.2 `timeSync` — the guides' payload is wrong, and this answers the clock question

`unifi-camera-reference.md` §13 asks how a real controller syncs the camera clock,
with NTP as the hypothesis. It is **not NTP**. The reply is a two-timestamp
exchange:

```json
ubnt_avclient_timeSync   {"t1": 1784981269413, "t2": 1784981269413}
```

The guides record the reply as carrying `monotonicMs`, `wallMs`, `wallClockMs`,
`time`, `seconds`, `timeDelta`. **None of those appear.** `t1`/`t2` is an
NTP-style offset exchange done in-band over `:7442`. It is also the single most
frequent message on the channel — 42 from the camera, 20 from the controller in
one session. [MEASURED]

### 9.3 Audio: it flows, it's AAC, and the lever is the *shape* of the audio object

`unifi-camera-reference.md` §8 concludes "a third-party controller cannot get
in-band audio" after an exhaustive table of failures, and §13 asks what sets
`av.audio.volume`. Both are now answered.

**Audio tags flow: ~950–1100 per stream, on all three channels.** Format from the
wire: `af 00` sequence header, `soundFormat=10` (AAC), AudioSpecificConfig
`objectType=2` (AAC-LC), **16000 Hz, mono**. [MEASURED]

What the real controller sends in `ChangeVideoSettings`:

```json
"audio": {"bitRate": 64000, "volume": 100}
```

**Two keys.** The guides' failed attempts sent a ten-key object — `bitRate`,
`channels`, `description`, `enableTemporalNoiseShaping`, `enabled`, `mode`,
`quality`, `sampleRate`, `type`, `volume` — and recorded that the camera
"force-resets `volume` to 0 in the echo, every time".

The real controller sends `volume: 100` in a minimal two-key object **and audio
works**. That makes the *shape* of the audio object the leading suspect, not the
codec and not `withOpus`. Testing a two-key `{bitRate, volume}` against a
third-party controller is the obvious next experiment. [MEASURED] / lever
[INFERRED]

### 9.4 Video codec id is **8**

Every video tag carries `codecId = 8` — never 7 (AVC) and never 12 (the
conventional HEVC-in-FLV id). The sequence header arrives with `frameType = 6`.
FLV flags byte is `0x07` as the guides record. **A receiver must map `8 → HEVC`;
no standard demuxer will.** [MEASURED]

### 9.5 The hello reply and `paramAgreement` are much smaller than documented

```json
ubnt_avclient_hello (controller -> camera)
  keys: protocolVersion, controllerName, controllerUuid, controllerVersion, overrideUuid

ubnt_avclient_paramAgreement (controller -> camera)
  {"enableStatusCodes": true, "useHeartbeats": false, "heartbeatsTimeoutMs": 60000}
```

The values matter as much as the key list:

```json
{"protocolVersion":67,"controllerName":"unifi-emu","controllerUuid":null,
 "controllerVersion":"7.1.77","overrideUuid":true}
```

Three things: **`overrideUuid` is actually sent** — as a boolean `true`, alongside
a `controllerUuid` of **`null`**, on every hello reply, not as the escape hatch the
guides describe. The camera's persisted-UUID comparison is therefore a path the
real controller never takes, which is what makes it possible to stand in for a
console without resetting the camera. Next: the hello reply carries **no
`adoptionCode` and no `features` block**; and `paramAgreement` carries **no
`authToken` and no `features`** — nothing like the
`{authToken, features, controllerUuid, uuid}` the guides record. Note
`useHeartbeats: false` with a 60 s timeout. [MEASURED]

### 9.6 The PTZ channel is a separate WebSocket with subprotocol `ptz1`

From `devices.websockets.log`:

```
onConnection() Received websocket connection wsproto=secure_transfer   isAvClient=true
onConnection() Received websocket connection wsproto=ptz1              isAvClient=false
```

So `EnablePtzControl{uri}` (§1) causes the camera to open a **second** WebSocket
to the same `/camera/1.0/ws` path, negotiating subprotocol **`ptz1`** instead of
`secure_transfer`. That completes the `PtzWebsockHandler` picture: the trigger,
the URI, and the distinguishing subprotocol. [MEASURED]

Also observed on that channel: `Preset` rejected with a non-zero status code, an
automatic *"Returning home after 30000ms"*, and `disablePtzControlMany` returning
`ENOSYS`.

### 9.7 A fingerprint check gates the connection

Before accepting either WebSocket, the controller runs:

```
verifyFingerprintHttp fingerprint=<20-byte colon-separated hex>
  -> Fingerprint matches camera.fingerprint
```

A SHA-1-length fingerprint compared against a stored per-camera value. **This
appears nowhere in the existing guides**, and any controller emulator will have to
satisfy or bypass it. [MEASURED]

### 9.8 Vocabulary not previously documented

| Message | Direction | Payload |
|---|---|---|
| `EventStreamChanged` | camera → controller | `{}` — frequent (26 in one session) |
| `EventLowMemoryState` | camera → controller | `{"clockMonotonic":…,"clockWall":…,"process":"audio_events","state":"enabled"}` |
| `SendWeatherUpdate` | **both directions** | controller pushes weather; camera responds |
| `ChangeClarityZones`, `ChangeInterfaceSettings`, `UpdateFaceDBRequest`, `ChangeSmartMotionSettings` | controller → camera | acked by the camera |

**`EventLowMemoryState` is quietly important**: it reports `process:"audio_events"`,
`state:"enabled"`. `unifi-camera-reference.md` §10 records `ubnt_audio_events` as
boot-disabled, which is why `ChangeAudioEventsSettings` was rated hazardous. On a
camera adopted by a real controller **that daemon is running** — and the camera
acks `ChangeAudioEventsSettings` normally.

### 9.9 Direction is explicit, and acks echo the verb

`from`/`to` make direction unambiguous — `ubnt_avclient` is the camera,
`UniFiVideo` the controller. Counting by direction confirms the guides' ack model:
`ChangeVideoSettings` appears **13× from the controller and 12× from the camera**,
because the camera's ack reuses the same `functionName`. Correlate on
name + `inResponseTo`, never on id alone. [MEASURED]

---

## 10. Controller internals worth knowing

Not protocol, but useful context for anyone reproducing this. All from the running
container. See [`techniques.md`](techniques.md) §7–8 for how.

**`ds` is the camera-facing daemon.** A Rust binary (`tokio-tungstenite`) serving
`:7442`, launched as `ds --config /usr/share/ds/ds.json` with
`EnvironmentFile=-/etc/ds/default` — that file does not exist by default and is
yours to create. Environment variables it honours include `LOG_LEVEL`,
`SSL_CERTIFICATE_VERIFICATION_ENABLED`, `SSL_ALLOW_SELF_SIGNED`,
`SSL_IGNORE_SNI_HOSTNAME`, `DS_RECORDING_DISABLED`, and a family of
`WEBSOCKET_*` tunables (connection timeout, max failed handshakes per minute,
reconnect blacklist thresholds).

**The controller manages camera firmware.** `/srv/unifi-protect/downloads/`
contained two `sav530q` images — `5.3.95` and `4.74.106` — so it fetches camera
firmware on its own initiative. Relatedly, `devices.websockets.log` reports the
camera's `firmwareVersion=4.74.106` at connection time, which does not match the
`5.3.95` the other guides record as the device-of-record firmware. Treat "the
camera's version" as ambiguous between firmware and middleware unless the source
is explicit.

**Ports `ms` listens on**, beyond the documented ones: `7441`, `7445`, `7446`,
`7447`, `7451`, plus `7550` and `7552`. `7552` listening is notable given the
per-track-port question in §3.1.

**`ds` computes recording gating itself.** Its strings include
`motion_only=false: no internal SSD, no /srv`,
`motion_only=true: has internal SSD, no /srv`, and
`Recording disabled via DS_RECORDING_DISABLED override` — a second, independent
gate alongside `unifi-protect.service`'s `UFP_RECORDING_*` environment
calculation.

**Vocabulary available from `ds`'s own type definitions** — far larger than what
adoption exercises. `CameraFeatureFlags` alone enumerates `hasLiveviewTracking`,
`hasLineCrossing`, `hasLineCrossingCounting`, `presetTour`, `hasSmartZoom`,
`hasOptimizeIr`, `streamEncryptable`, `supportFullHdSnapshot`,
`hasTamperDetection`, `clarityZones`, `hasHallwayMode`, `hasPackageZoneSupport*`,
`smartDetectAudioTypes`, `videoCodecs`, `audioCodecs`, `audioStyle` and dozens
more. `struct CameraMessage {functionName, inResponseTo, messageId, timeStamp, …}`
confirms the envelope shape from the other side. Anyone extending `cuckoo`'s
`features` block should mine this list rather than guess.

---

## 11. API → wire mapping — `[MEASURED]`

Driving the controller through a third-party client while capturing `:7442` in
plaintext gives a direct mapping from *what you ask the controller to do* to *what
it says to the camera*. Method in [`techniques.md`](techniques.md) §13.

**Client:** `uiprotect` 15.14.2 (Python), authenticated as a local admin.
**Capture:** `ds` payload logging (§9.1), frames reassembled and split by the
`from` field.

| API call | Controller sends |
|---|---|
| `set_osd_name`, `set_osd_date`, `set_osd_logo`, `set_osd_bitrate` | `ChangeOsdSettings` |
| `set_status_light` | `ChangeSoundLedSettings` |
| `set_video_mode` | `ChangeVideoSettings` |
| `set_mic_volume` | `ChangeVideoSettings` |
| **`set_ssh`** | **`StartService`** |
| `set_person_detection`, `set_person_track` | `ChangeAudioEventsSettings`, `ChangePTZAutoTrackSettings`, `ChangeSmartDetectSettings` |
| `set_recording_mode` | `ChangeSmartDetectSettings`, `ChangeSmartMotionSettings` |
| `set_privacy` | `ChangeIspSettings` |
| `set_vehicle_detection` | *(nothing — controller-side state only)* |

### 11.1 `StartService` — this answers "what re-enables SSH?"

`unifi-camera-reference.md` §4 records SSH as off by default and enabled "from a
controller that has adopted the camera", with the mechanism `[UNVERIFIED]`. §9.8
found the controller sending `StopService {"service":"ssh"}` during adoption. The
counterpart is now confirmed:

```
cam.set_ssh(True)   ->   StartService     (controller -> camera, :7442)
```

**`StartService` appears nowhere in the guides' vocabulary.** Together with
`StopService` it is a general service-control verb, which suggests other
boot-disabled daemons — `ubnt_audio_events`, `ubnt_ptz` — may be startable the
same way. That is worth testing: `unifi-camera-reference.md` §13 asks what
re-enables `ubnt_ptz` given `rc.sysinit:523` disables it.

### 11.2 One API call fans out to several messages

`set_person_detection` and `set_person_track` each produce **three** messages.
An emulated controller that sends only `ChangeSmartDetectSettings` when enabling
person detection is not reproducing what a real one does.

Note `ChangeAudioEventsSettings` among them, **acked normally**. That is the third
independent confirmation that `protocol-reference.md` §6's "hazardous, tears down
the control channel" rating was an artefact of testing against a camera whose
`ubnt_audio_events` daemon was boot-disabled — see §9.8.

### 11.3 Privacy masking is an ISP operation

`set_privacy` produces `ChangeIspSettings`, not a dedicated message. The privacy
mask is imaging-pipeline state, consistent with zoom and focus belonging to
`ubnt_ispserver` rather than the gimbal.

### 11.4 Mic volume rides in `ChangeVideoSettings`

`set_mic_volume` produces `ChangeVideoSettings` — the message whose `audio` block
is `{bitRate, volume}` (§9.3). So **`set_mic_volume` is the API path that sets the
volume field**, which `unifi-camera-reference.md` §13 asks about. It does not by
itself prove what a third-party controller must send, but it identifies the
operation to watch when testing the two-key audio object.

### 11.5 Not everything reaches the camera

`set_vehicle_detection` produced **no `:7442` traffic at all** — the controller
records it and acts on it during its own analysis. Useful negative information:
an emulator does not need to implement everything the API exposes, and a message
absent from a capture may simply never have existed.

---

## 12. Client-driven sweep — `[MEASURED]`

A systematic pass with both API clients, capturing `:7442` throughout. Method in
[`techniques.md`](techniques.md) §13.

### 12.1 `ChangeIspSettings` — the documented payload is substantially wrong

The real message carries **36 fields**. `protocol-reference.md` §4 documents 42.
**Only 21 overlap.**

**Sent but undocumented (15):**

```
autoFlipMirror  colorNightVision  dZoomCenterX  dZoomCenterY  dZoomScale
dZoomStreamId   enableExternalIr  hdrMode       icrCustomValue  icrSwitchMode
lensDistortionCorrection  masks  smokeCoverMode  spotlightDuration  zonesAutoFlipMirror
```

**Documented but never sent (21):** `aeTargetPercent`, `criticalTmpOfProtect`,
`darkAreaCompensateLevel`, `enableMicroTmpProtect`, `forceFilterIrSwitchEvents`,
`icrLightSensorNightThd`, `queryIrLedStatus`, and **the entire `irOnSts*` and
`irOnVal*` families** (12 fields).

⚠️ **`aeTargetPercent` is among the missing.** `unifi-camera-reference.md` §7
builds an exposure table on it — that table may still be valid for a *directly
driven* camera, but a real controller does not use that field. Anything relying on
the `irOnSts`/`irOnVal` families should be treated as unverified.

`masks` confirms §11.3: privacy masking is carried inside `ChangeIspSettings`.
`dZoom*` fields indicate digital zoom, consistent with this model reporting
`canOpticalZoom: false`.

### 12.2 `ChangeAudioEventsSettings` — schema validated, and trimmed

The recovered schema in `unifi-camera-reference.md` §10 is **correct about types
and wrong about size**. The real payload is **13 keys**:

```json
{"enableAlrmBabyCry":1,"enableAlrmBark":0,"enableAlrmBurglar":0,"enableAlrmCarHorn":0,
 "enableAlrmCmonx":1,"enableAlrmGlassBreak":0,"enableAlrmSiren":0,"enableAlrmSmoke":1,
 "enableAlrmSpeak":1,"recordEventCmonx":0,"recordEventSmoke":0,"recordEventSpeak":0,
 "sendPulse":0}
```

- ✅ **Values are integers `0|1`, never booleans** — the guides' claim is confirmed
  on the wire.
- ❌ **No thresholds, no lingers, no level bounds, no `pulsePeriodSec`.** The
  documented `thresholdEvent*`, `linger*StartSec/StopSec`, `smokeThreshold`,
  `cmonxThreshold`, `levelUpperBound`, `levelLowerBound`, `leveldBShifter` are
  **not sent**.
- ❌ `enableLoudNoise` and `enableSoundLoss` are documented but absent.
- Only **three** `recordEvent*` keys exist (Cmonx, Smoke, Speak) — not one per
  class. Notably there is no `recordEventBabyCry` even when baby-cry is enabled.
- The controller sends **all nine** `enableAlrm*` keys even though this camera
  advertises only four supported classes (`alrmSmoke`, `alrmCmonx`, `alrmBabyCry`,
  `alrmSpeak`).

**And it is safe.** Driving it repeatedly produced clean acks every time. That is
now the fourth independent confirmation that §6's "hazardous" rating was an
artefact of a boot-disabled daemon.

### 12.3 Talkback — the settings, without needing a speaker

`ChangeTalkbackSettings` is `[UNVERIFIED]` throughout the guides. The bootstrap
exposes the live values even on a camera reporting `hasSpeaker: false`:

```json
{"typeFmt":"aac","typeIn":"serverudp","bindAddr":"0.0.0.0","bindPort":7004,
 "filterAddr":null,"filterPort":null,"channels":1,"samplingRate":22050,
 "bitsPerSample":16,"quality":100}
```

**Talkback is AAC over UDP on port 7004**, mono, 22.05 kHz, 16-bit. `typeIn`
`serverudp` indicates a bound UDP listener rather than the outbound-TCP pattern
the media channels use.

⚠️ **Port 7004 is not in any port table in these guides.** Add it.

Driving talkback is refused client-side (`Camera does not have speaker`) before
any request is issued, which likely explains why this was never mapped.

### 12.4 `StartService` / `StopService` are a symmetric pair

```json
StopService   {"service": "ssh"}
StartService  {"service": "ssh"}
```

A service-name field in both directions. That the verb is parameterised — rather
than an `EnableSsh`-style dedicated message — is the interesting part: it suggests
**any service name may be startable**, which bears directly on
`unifi-camera-reference.md` §13's question about what re-enables `ubnt_ptz`.

**Not yet tested**, because no client exposes an arbitrary service name; it needs
a controller that can send a chosen payload — i.e. `cuckoo`.

### 12.5 The realtime bus is a delta stream, not a relay

The controller's realtime API carries `{header, payload}` packets:

| header | payload |
|---|---|
| `action=add modelKey=event` | `type, start, locked, isFavorite, favoriteObjectIds, score` |
| `action=update modelKey=nvr` | `lastSeen, wanPorts, portStatus` / `systemInfo` |
| `action=update modelKey=automation` | `status, cooldown` |

`payload` carries **only changed fields**, keyed by `modelKey` and `id`.

**Camera events are not relayed verbatim.** The camera sends `EventSmartDetect` on
`:7442`; the controller *synthesises* a Protect `event` record with fields the
camera never sent — `score`, `locked`, `isFavorite`, `favoriteObjectIds`. So a
`lyrebird` that emits camera-side events gets Protect-side records built for it,
and a `cuckoo` consuming camera events must construct these itself.

### 12.6 Codec and audio — independently confirmed

§9.3 and §9.4 rested on one packet capture parsed by one hand-written FLV walker.
Re-checked through a completely separate path — hjdhjd's livestream API, which
returns fMP4 rather than `extendedFlv` — and read with `ffprobe`:

```
codec_name=hevc   video   2688x1512
codec_name=aac    audio   16000 Hz   1 channel
```

**H.265 confirmed. AAC-LC 16 kHz mono confirmed.** Two independent transports, two
independent parsers, same answer. The fMP4 `stsd` carries `hvc1`, which is the
conventional HEVC fourcc — so Ubiquiti's non-standard `codecId 8` (§9.4) is a
quirk of *their* FLV container only; the controller re-muxes to standard fMP4 on
the way out.

### 12.7 Feature flags — the authoritative list for `cuckoo`

Protect's bootstrap reports **57** flags; `ds`'s `CameraFeatureFlags` struct
defines **68** names. **39 appear in both** — that intersection is what a camera
emulator should be prepared to advertise:

```
audioCodecs audioStyle canAdjustIrLedLevel canMagicZoom canOpticalZoom
canTouchFocus hasAec hasBluetooth hasColorLcdScreen hasExternalIr
hasFingerprintSensor hasHdr hasIcrSensitivity hasInfrared hasLcdScreen hasLdc
hasLedStatus hasLineCrossing hasLineCrossingCounting hasLineIn
hasLiveviewTracking hasMic hasMotionZones hasPrivacyMask hasRtc hasSdCard
hasSpeaker hasSquareEventThumbnail hasVerticalFlip hasWifi isDoorbell isPtz
lensModel mountPositions smartDetectAudioTypes supportFullHdSnapshot supportNfc
videoModeMaxFps videoModes
```

29 exist only in `ds` (the controller understands them; this camera does not
advertise them) — including `clarityZones`, `excludeZones`, `hasSmartZoom`,
`hasTamperDetection`, `presetTour`, `streamEncryptable`, `videoCodecs`.

Note `pan`, `tilt`, `zoom`, `focus` and `hotplug` appear on the Protect side as
**nested objects**, matching the `features.pan.steps` shape in §3 — they are not
booleans.

---

## 13. PTZ, fully mapped — `[MEASURED]`

Driving `POST /proxy/protect/integration/v1/cameras/{id}/ptz/goto/{slot}` against
four presets, capturing `:7442` throughout. This closes the largest remaining gap.

### 13.1 `Preset` is a two-step, action-based verb — and `AbsolutePosition` is never used

```json
Preset {"action":"config",
        "items":[{"index":1,"name":"Preset 2","pan":23502,"tilt":8000,"zoom":0,"focus":59}]}

Preset {"action":"go","index":1,"speed":1000,"notifyCommandStatus":{}}
```

1. **`config`** pushes the preset *definition* — index, name, and raw motor
   `pan`/`tilt`/`zoom` **plus `focus`**.
2. **`go`** executes it by index, with a `speed` and `notifyCommandStatus`.

**The controller does not send `AbsolutePosition` at all.** A move to arbitrary
coordinates is expressed as *configure a preset, then go to it*. That reframes
`protocol-reference.md` §8 completely: the reason `AbsolutePosition` looked like
the only thing that moved the gimbal is that it was being sent over SSH/IPC to
`ubnt_ptz` directly — the daemon's own interface. **Over `:7442`, the controller's
vocabulary is `Preset`.**

Note this is the *same verb* seen during adoption with
`{"trackTimeoutSec":20,"backToPresetPosition":true}` (§2) — so `Preset` is
multiplexed by `action`, and that adoption-time message is a third action
configuring auto-tracking return behaviour. Our earlier "rejected `Preset`" was a
malformed call, not an unsupported one.

`speed: 1000` here, against the `500 = saturation` measured for `AbsolutePosition`
over IPC — the scales are not the same and should not be assumed equivalent.

### 13.2 `GetCurrentPosition` **is** polled — position is not broadcast-only

The controller sends `GetCurrentPosition {"inDegree":true,"inSteps":true}`
**repeatedly** — before the move, during it, and several times after it settles.

`unifi-camera-reference.md` §9 states position "cannot be *queried*, you
subscribe" and that `GetCurrentPosition` times out. **On `:7442` it is a
polled request/response**, and the controller relies on it. The earlier timeout
was an IPC-path artefact, and the payload shape is what made it work.

**Both mechanisms are live at once**: the controller polls *and* the camera
broadcasts — 80 `EventMotorState` messages for a single preset move.

### 13.3 `EventMotorState` carries more than documented

```json
{"ignoreActivity": true,
 "state": {"activity": 16, "focusMode": "manual", "scale": "normalized",
           "position": {"focus": 58, "pan": 23502, "tilt": 8000, "zoom": 0},
           "wallClockMs": 1784989188609}}
```

New against the guides' recorded shape: **`ignoreActivity`**, **`focusMode`**, and
**`scale: "normalized"`** — the last implying position may be reported in more
than one coordinate system, which a consumer must not assume. `activity: 16` is a
value not previously seen (§9 records 0 settled, 12 moving, 8 elsewhere); treat it
as a flag word, not a magnitude, as the guides already advise.

The final broadcast's `position` matches the requested preset exactly
(`pan 23502, tilt 8000` for Preset 2), so it is a reliable completion signal.

### 13.4 `GetRequest` is the snapshot upload trigger — and it exposes port 7444

`protocol-reference.md` §9 lists `GetRequest` as "believed a snapshot-upload
trigger; shape unconfirmed". Confirmed:

```json
GetRequest {"what": "snapshot",
            "uri": "https://<controller>:7444/internal/camera-upload/<token>",
            "timeoutMs": 60000,
            "quality": "medium"}
```

The controller hands the camera a **one-time upload URL** and the camera **pushes
the image back** — the inverse of the pull model one might assume. Each request
carries a fresh opaque token.

⚠️ **Port 7444/TCP is the camera-upload endpoint**, path
`/internal/camera-upload/<token>`. Ubiquiti's own adoption documentation lists
7444 without saying what it does; this is what it does. Any controller emulator
wanting snapshots must serve it.

`quality` is an enum — `"medium"` observed.

### 13.5 What this means for an emulator

- Implement `Preset` with `action: config` / `action: go`, not `AbsolutePosition`.
- Serve **:7444** with a token-scoped upload endpoint, and drive snapshots with
  `GetRequest`.
- Poll `GetCurrentPosition` *and* consume `EventMotorState`; the real controller
  does both.
- Preset definitions include **focus**, so a four-axis model (pan/tilt/zoom/focus)
  is the right shape — not the three-axis one in cuckoo.old's `ptz-model.md`.

---

## 14. The camera side, measured by being one — `[MEASURED]`

Everything above watches the controller. This section is the other direction:
what a controller demands of a **camera**, learned by writing one
([`finch`](https://github.com/rjmotion)) and getting it adopted, streaming and
recorded by a real Protect 7.1.77.

### 14.1 The adoption token is one API call

```
POST /api/auth/login                            → session cookie
GET  /proxy/protect/api/cameras/manage-payload  → {"mgmt": {...}}
```

```json
{"wifi": {}, "mgmt": {"protocol": "wss", "hosts": ["<CONTROLLER-IP>:7442"],
                      "token": "<32-CHAR-TOKEN>"}}
```

The token is nested under `mgmt`, **not** at the top level, and lasts an hour. The
camera quotes it twice: in the WebSocket URL as `?token=…` and in its hello as
`adoptionCode`.

### 14.2 The camera's handshake headers are load bearing

`ds` does not terminate the camera connection — it **proxies** it to
`ws://127.0.0.1:7448/ws`, forwarding the camera's headers. Get them wrong and the
visible symptom is `HTTP error: 400 Bad Request` in `ds.log` with the socket
closed immediately after the upgrade appears to succeed.

What the real G5 sends:

```
GET /camera/1.0/ws HTTP/1.1
Host: <CONTROLLER-IP>              ← no port
Pragma: no-cache
Cache-Control: no-cache
Upgrade: websocket
Connection: close, Upgrade         ← not plain "Upgrade"
Sec-WebSocket-Key: …
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: secure_transfer
Origin: http://ws_camera_proto_secure_transfer
camera-mac: <CAMERA-MAC>
camera-ip: <CAMERA-IP>
camera-model: 0xa59b               ← the hex system id, NOT "UVC G5 PTZ"
camera-firmware: 5.3.95
device-id: <UUID>                  ← stable per device
x-guid: <UUID>                     ← per connection
adopted: true|false
```

`camera-model` as a model *name* is enough on its own to be refused upstream.

### 14.3 Nulls in a settings message are questions, not values

`ChangeVideoSettings` arrives with `"fps": null`, `"bitRateVbrMax": null`,
`"minClientAdaptiveBitRate": null` and `"enabled": false`. These are not settings
to apply — they are the controller asking *what are you?*. A camera that echoes
the nulls back has them **stored verbatim**, and the controller then believes it
has a camera whose channels have no frame rate, no bitrate and are disabled — so
it never asks for video and no stream is ever armed.

Report real values instead, with `enabled: true` and the codec, and the controller
immediately follows with a `ChangeVideoSettings` carrying `destinations`. This one
behaviour is the difference between an adopted camera that does nothing and a
working one.

### 14.4 `onMetaData` is an AMF0 **object** with exactly nine keys

Captured from the real camera's first tag on `:7550`:

```
audioBandwidth  = 64000      audioChannels = 1        audioFrequency = 16000
channelId       = 0          extendedFormat = true    hasAudio = true
hasVideo        = true       streamId = 1             streamName = "<16-CHAR-ALIAS>"
```

Two traps. The value is an AMF0 **object (`0x03`)**, where ffmpeg and most FLV
writers emit an ECMA array (`0x08`). And the key set is nothing like ffmpeg's —
there is no `width`, `height`, `framerate`, `videocodecid` or `duration`; the
receiver takes geometry from the bitstream.

**`channelId` is how the stream is identified.** Recordings are filed as
`<MAC>_<channelId>`, and the media server logs
`NO INPUT STREAM <MAC>_0 FOR RECORDING AVAILABLE YET` until a stream announces the
matching id. `streamName` is the per-session alias the controller handed out in
`avSerializer.parameters.streamName`.

Channel map on the G5 PTZ: `video1` → channelId 0, streamId 1 · `video2` → 1, 2 ·
`video3` → 2, 4.

### 14.5 The 16-byte trailer, decoded

The guides recorded its length; this is its content, confirmed against
unifi-cam-proxy's `clock_sync.py` and our own capture:

```
00              one zero byte
01 5F 90        0x015F90 = 90000   on video tags — the clock rate
00 2B 11        0x002B11 = 11025   on everything else
00 × 8          padding
uint32          elapsed seconds × 100000
```

The real camera also emits `onClockSync` (`streamClock`, `streamClockBase`,
`wallClock`) and `onMpma` (a bitrate envelope) as script tags every few seconds.

### 14.6 The media server talks back on `:7550`

The push socket is not one-way. Once a stream is linked, the receiver sends short
TLV messages — 2-byte type, 2-byte length, then the body:

```
00 00 00 0c ff ff ff ff 00 00 00 00 00 ff 00 00     type 0, 12 bytes
00 02 00 03 00 01 01                                type 2, 3 bytes
00 02 00 03 80 00 01                                type 2, 3 bytes
00 01 00 00                                         type 1, empty
```

Type 2 alternates between two bodies and arrives every few seconds, which is
consistent with a stream-state or keepalive instruction. **Not yet decoded** — but
a camera implementation that never reads its push socket will not notice these,
and will not notice the receiver closing either. [UNVERIFIED beyond the framing]

### 14.7 Success looks like this

With the above corrected, a purely synthetic camera reaches
`RECORDING STARTED <MAC>_2` in the media server's own log, having been discovered,
adopted, configured and streamed to entirely over the real protocol — with
**H.265**, which no existing camera-emulation project sends.

---

## 15. Custody — what it takes to move a camera between controllers — `[MEASURED]`

Taking the socket is not taking the camera. Measured by pointing a replacement
controller at a real, already-adopted G5 PTZ:

### 15.1 An adopted camera says almost nothing

Stop the resident controller, bind `:7442` yourself, and the camera reconnects
within about 90 seconds — with `adopted: true` in its upgrade headers. Then:

- it sends **`ubnt_avclient_timeSync` and nothing else**, roughly every 60s
- it never sends a hello. The hello is how a camera *introduces itself to a new
  controller*, and as far as this one is concerned it already has one
- settings sent to it come back **`{"desc": "Unauthorized"}`**
- an **unprompted hello from the controller is ignored** — no reply, no error, and
  the connection is dropped at its own timeout. `overrideUuid` applies to a hello
  the camera asked for, not to an introduction it did not

So the `controllerUuid: null` + `overrideUuid: true` pair (§9.5) is *not* a way to
steal an adopted camera. It is how a controller answers a camera that is already
asking to be adopted.

### 15.2 A released camera stops dialling entirely

Ask the resident controller to release it — an ordinary unadopt, which needs no
access to the camera — and its behaviour changes completely:

- it **stops connecting to `:7442`** altogether
- it starts answering discovery probes on **UDP 10001** again, ~178 bytes each
- it is waiting to be *told where to go*: the controller reaches it over HTTPS on
  port 443 with the adoption details (`POST /api/1.2/manage`), and only then does
  it dial `:7442` and say hello

This is the same native flow a factory-fresh camera follows, and it means a
replacement controller needs that HTTPS step to take custody — the wire side alone
is not enough.

### 15.3 The practical shape of a handover

| Direction | What has to happen |
|---|---|
| **To a replacement controller** | resident controller releases the camera → replacement discovers it on UDP 10001 → replacement POSTs `/api/1.2/manage` → camera dials `:7442`, says hello, ordinary adoption |
| **Back to the resident** | replacement stops → resident controller adopts it again through its own API → camera returns to `CONNECTED` |

The second direction is one API call and was verified end to end. The first is now
implemented — see §15.5 — and needs one thing that is not ours to invent.

### 15.5 The manage handover, decoded

The `/api/1.2/manage` POST — the step after discovery — is two requests, read from
the camera side (unifi-cam-proxy) and confirmed against the controller side
(Protect makes 208 of these while adopting):

```
POST https://<camera>/api/1.2/login    {"username":…, "password":…}
    → 200, Set-Cookie: TOKEN=<hex>
POST https://<camera>/api/1.2/manage    {"mgmt": {"token":…, "hosts":[…], "protocol":"wss"}}
    with that cookie
    → 200; the camera then dials hosts[0] and says hello
```

Two asymmetric facts:

- **The manage `token` is the controller's to choose.** It becomes the
  `adoptionCode` the camera quotes back in its hello, and the controller is the
  thing that decides whether to accept it. It need not come from a real console.
- **The login credential is the *camera's*, and cannot be chosen.** It is the
  default (`ubnt`/`ubnt`) only on a factory-fresh or reset device; otherwise it is
  whatever controller adopted the camera last set and stored. Unauthenticated, and
  with a wrong credential, the endpoint returns a bare `401` with no
  `WWW-Authenticate` header.

So a replacement controller can point a camera at itself — but only with the
camera's current management credential, which the previous controller holds. There
is no wire-level way around this; custody is ultimately a shared secret, and
handing it over is a decision for whoever owns the camera. Releasing the camera
(§15.2) does **not** reset that credential.

### 15.4 An already-adopted camera is not a bug to work around

Worth stating plainly, because the temptation is to keep trying: there is no
message that takes an adopted camera away from its controller. Every attempt above
was met with silence or `Unauthorized`. Custody is granted by the controller that
holds it, and that is the mechanism to use.

---

## 16. cuckoo drives the real camera — the last two gates — `[MEASURED]`

The camera-side impersonation (finch) worked long before the controller-side one
(cuckoo) could drive real hardware. Two gates stood in the way, both found by
capturing a real adoption in plaintext (`ds` trace, §10) and comparing it to what
cuckoo did.

### 16.1 timeSync is a ping-pong, and the controller must answer every one

The camera does **not** say hello on connect. It opens, and sends
`ubnt_avclient_timeSync` — then keeps sending it, roughly ten times, each message
carrying `inResponseTo` referencing the controller's *previous* timeSync. The
controller answers every one:

```
camera →  timeSync  inResponseTo=0            "how wrong am I?"
       ←  timeSync  inResponseTo=<cam msgId>  {"t1":<ms>,"t2":<ms>}
camera →  timeSync  inResponseTo=<ctrl msgId> "how wrong am I now?"
       ←  timeSync  inResponseTo=<cam msgId>  {"t1":<ms>,"t2":<ms>}
   … about ten round trips, until the camera's clock converges …
camera →  hello     inResponseTo=0            adoptionCode="", full features
```

Only after that exchange does the camera introduce itself — **unprompted**, with
an empty `adoptionCode`. The `{"t1","t2"}` reply payload (§9.4) is exactly right,
and identical to what the real controller sends.

The trap: a timeSync after the first carries `inResponseTo`, so a controller that
treats "has inResponseTo" as "this is a reply, ignore it" answers only the first
and goes silent. The ping-pong dies, the clock never converges, and the camera
sits sending timeSync forever without ever reaching hello. **Answer every
timeSync, reply flag or not.**

### 16.2 The controller does not speak first

It is tempting, when a camera connects and only pings timeSync, to send it a hello
to get things moving. The real controller never does — it waits. An unprompted
controller hello (with or without `overrideUuid`) is **ignored**: the camera does
not reply to it and does not advance. The hello is the camera's to send, once its
clock is set. (An older cuckoo did send an initiating hello and adopted anyway,
which sent us down this path; against this firmware, waiting is correct.)

### 16.3 Result

With both corrected, cuckoo — pointed at the camera by its own `/api/1.2/manage`
POST (§15.5) — completed a real adoption: ten timeSync round trips, the camera's
hello, the settings suite acknowledged, `EnablePtzControl` accepted, motor bounds
read from the hello (`pan 500..35500, tilt 8000..18000`), and the camera pushing
**H.265 to cuckoo's ingest** — 37 MB in the first minute. The first time the
controller side has driven real hardware rather than a fake camera.

### 16.4 The real camera's sequence header is not an hvcC — RESOLVED

The RTSP 503 was not a framing problem: the `extendedFlv` tag boundaries are exactly
as documented (20 bytes between tags, every gap confirmed against a real capture).
It was the *video sequence header* itself. Two differences from what a standard
FLV/HEVC writer produces, both measured off a real G5 PTZ:

- **The FLV/AVC packet-type byte is `1`, not `0`, even on the config tag.** A
  reader that keys on it — as the AVC convention says — treats the parameter sets
  as an ordinary frame and never parses them. **Config is signalled only by the
  FLV frame type (`6`) in the high nibble of byte 0.**
- **There is no hvcC record.** After the two-byte header, the sets are a bare run
  of **2-byte length-prefixed** NAL units: `[len]VPS[len]SPS[len]PPS`, no
  configuration prefix and no array structure. (The frame NALUs that follow use
  **4-byte** lengths, as a normal hvcC would specify.)

```
video sequence-header tag body:
  68            FLV: frame type 6 (sequence header), codec id 8 (HEVC)
  01            packet-type byte — 1, where the AVC convention says 0
  00 18         length 24
  40 01 …       VPS (24 bytes)
  00 2a         length 42
  42 01 …       SPS (42 bytes)
  00 07         length 7
  44 01 …       PPS (7 bytes)
```

With config detected by frame type and the sets read from this layout, cuckoo
re-served the **real camera's H.265 as standard RTSP**: `ffprobe` reports
`hevc / Main / 2688x1512 / yuv420p / 30 fps` plus an AAC-LC track, and `ffmpeg`
decoded a full 2688×1512 picture out of it. The controller side now works end to
end against real hardware — adoption, PTZ, and video.
