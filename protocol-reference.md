# UniFi camera ↔ controller — wire protocol reference

The protocol a UniFi camera speaks to a UniFi Protect controller: transport,
envelope, adoption handshake, settings vocabulary, streaming containers, and
events. Companion to `unifi-camera-reference.md`, which covers the camera *as a
device* (OS, daemons, IPC, SSH, failure modes).

**Device of record:** UVC G5 PTZ, firmware 5.3.95 / middleware v4.70.91.

**Confidence:** `[M]` measured on hardware · `[F]` firmware analysis · `[I]`
inferred · `[U]` unverified.

---

## 1. Transport

### Ports

| Port | Proto | Direction | Purpose |
|---|---|---|---|
| **7442** | TCP/TLS | camera → controller | Management channel: WebSocket + JSON envelopes |
| **7550** | TCP plain | camera → controller | H.264 push, extendedFlv. **All H.264 tracks share this one port** |
| **7551** | TCP plain | camera → controller | MJPEG push |
| **10001** | UDP | camera → broadcast | Discovery |
| 22 / 443 | TCP | controller → camera | SSH; the camera's own local API |

The camera listens only on 22/80/443/10001. **Every media and control channel is
the camera dialling out** to a destination the controller hands it. [M]

### TLS

**The camera does not validate the controller's certificate.** A self-signed EC
(`prime256v1`) cert is accepted. This is the single most load-bearing fact about
the transport. Client-cert verification is not required for adoption. [M]
Camera-side cert material lives at `/etc/persistent/server.pem` and
`/etc/persistent/controller.crt`. [F]

### WebSocket upgrade

```
GET /camera/1.0/ws?token=<adoption-token> HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: <base64>
Sec-WebSocket-Protocol: secure_transfer
camera-mac: <hwaddr>
camera-model: <model>
camera-firmware: <version>
adopted: <true|false>
```

- Path is **`/camera/1.0/ws`**; the adoption token rides as `?token=`. [M]
- The camera offers subprotocol **`secure_transfer`**; the controller echoes it. [M]
- The four `camera-*`/`adopted` headers are its pre-handshake self-identification. [M]
- Controller replies `101` with standard
  `Sec-WebSocket-Accept = base64(sha1(key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))`. [M]
- Camera-side implementation is **libwebsockets**. [F]

### Framing and keepalive

- Standard RFC 6455. Camera→controller frames are **masked**; the reverse are not.
  JSON envelopes ride in **binary frames (0x2)**; text is also accepted. [M]
- A robust parser should try a raw UTF-8 JSON decode, then fall back to scanning
  for the first `{` — in practice this firmware sends bare JSON. [M]

> **The controller MUST proactively send WebSocket pings.** A controller that only
> *answers* the camera's pings and never *sends* one is judged dead: the camera
> resets the channel with `Connection reset by peer` ~5 s later and re-runs the
> entire adoption every ~6 s. **A ping every 2 s holds a connection stable.** [M]

Losing the control channel tears down video:
`Going to reset video destinations due to websocket ping-pong timeout to avoid
bandwidth competition`. [F]

Tunables in the binary: `heartbeatsTimeoutMs`, `timeSyncTimeoutMs`,
`waitConnectionTimeoutMs`, `useHeartbeats`. Backpressure guards:
`Too many enqueued WS messages`, `Abnormal # of enqueued WS requests`. [F]

### Pointing a camera at a controller

```
POST /api/1.0/login    {"username":"ubnt","password":"<pw>"}   -> session cookie
PUT  /api/1.1/settings {"controller":{"addr":"<ip>","token":"<token>"}}
GET  /api/1.1/status                                            -> {"controller":{...}}
```

Over the camera's own HTTPS API (self-signed, verification off). The camera
persists the adoption code in systemcfg under `unifivideo.adoptioncode`. [M]/[F]

---

## 2. The envelope

```json
{
  "from": "ubnt_avclient",
  "to": "UniFiVideo",
  "functionName": "ubnt_avclient_hello",
  "messageId": 12,
  "inResponseTo": 0,
  "payload": { },
  "responseExpected": true,
  "timeStamp": "2000-01-01T00:11:01.856+00:00"
}
```

| Field | Semantics |
|---|---|
| `from` / `to` | Camera is **`ubnt_avclient`**, controller is **`UniFiVideo`**. Replies invert them. |
| `functionName` | The verb. **A reply echoes the request's `functionName`** — correlate by *name + `inResponseTo`*, not id alone. |
| `messageId` | Monotonic per sender. Any integer; no negotiated range. |
| `inResponseTo` | `0`/absent = request. Non-zero = reply, carrying the peer's `messageId`. |
| `responseExpected` | Sender wants a reply. |
| `timeStamp` | ISO-8601 with ms and offset. The camera's value reveals its clock. |

Other keys in the binary's table: `delayResponseTo`, `requestId`,
`transactionId`, `isBroadcast`, `statusCode`, `queueIfDestUnavailable`. [F]

**Ack semantics:** a settings request is acknowledged by a camera→controller frame
whose `inResponseTo` equals the controller's `messageId`. **Several verbs are never
acked even when processed** — see §6. [M]

Dispatch failure strings: `Dropping unknown request '%s' from '%s'`,
`Cannot handle request '%s'`, `Cannot find service that handles '%s'`,
`Not yet authenticated to process request '%s'`. [F]

---

## 3. The adoption handshake

```
  camera                                        controller
  ──────                                        ──────────
  TCP :7442 → TLS (cert NOT validated)
  GET /camera/1.0/ws?token=… + camera-* headers
                                          ──►   101 Switching Protocols
                                                (echo Sec-WebSocket-Protocol)

  ubnt_avclient_timeSync {timeDelta:0}    ──►
                                          ◄──   reply timeSync
                                                  {monotonicMs, wallMs, wallClockMs,
                                                   time, seconds, timeDelta}
                                          ◄──   request ubnt_avclient_hello
                                                  {controllerName:"UniFiVideo",
                                                   controllerVersion:"1.21.4",
                                                   protocolVersion:67,
                                                   controllerUuid, uuid}

  ubnt_avclient_hello (device announce)   ──►
                                          ◄──   1. REPLY to that hello   ◀══ THE GATE
                                                  {protocolVersion:67, controllerName,
                                                   controllerVersion, adoptionCode,
                                                   features:{…}, controllerUuid, uuid}
                                          ◄──   2. ubnt_avclient_paramAgreement
                                                  {authToken, features, controllerUuid, uuid}
                                          ◄──   3. settings suite, ONE AT A TIME,
                                                    each released on the previous ack

  acks each settings message              ──►
  emits Event* unprompted                 ──►   ◀══ adopted:true
  opens :7550 / :7551 pushes              ──►
```
[M]

### What actually gates adoption

**Replying to the camera's own hello**, with `inResponseTo` = that hello's
`messageId`. Until then the camera just retransmits hello every few seconds and
ignores `paramAgreement` and the entire settings suite. [M]

**The token is not the gate.** The camera generates its own `authToken` in its
`paramAgreement` reply and adopts even with **no `?token=`** in the URL. [M]

### The controller UUID is load-bearing

The camera **persists the controller UUID it adopted under** and compares on
reconnect: `UUID mismatch: has %s, received %s`. On mismatch it refuses to proceed
past hello — no settings acks, no video — and the failure presents as "adoption
looks fine but nothing streams". **Keep the UUID stable across restarts.** Sending
it under both `controllerUuid` and `uuid` is safe. `overrideUuid` is the
documented escape hatch (`Controller requested UUID override`), unexercised. [M]/[F]/[U]

### timeSync

The camera sends `{"timeDelta": 0}` while its envelope `timeStamp` carries its
real clock — it is asking *how wrong am I?* The reply carries `monotonicMs`,
`wallMs`, `wallClockMs`, `time`, `seconds`, `timeDelta`.

**Measured negative:** this camera **does not apply time from the reply** — its
`timeStamp` stays at the power-on epoch across every sync, and the burned-in OSD
date keeps reading 2000-01-01. A real controller almost certainly syncs by **NTP**
(`ntpClientHost` is a `ChangeNvrSettings` neighbour). [M]/[I]

### The hello — the camera announces its own limits

`features` is a nested capability object containing at minimum:

```json
"features": {
  "pan":  {"steps": {"min": 500,  "max": 35500}},
  "tilt": {"steps": {"min": 8000, "max": 18000}},
  "zoom": {"steps": {"min": 0,    "max": 730}},
  "mic": 1,
  "smartDetect": ["person","vehicle","animal","liveviewTracking",
                  "autoTracking","alrmSmoke","alrmCmonx","alrmBabyCry"],
  "audioCodecs": ["aac","opus"], "audioStyle": ["nature","noiseReduced"]
}
```

Also: `fwVersion`, `semver`, `uptime`, `hwaddr`, `hwrev`, `sysid`, `lensmodel`,
`cameraName`, and generation flags `isGen4c/4l/4s/4v/5s`, `isDoorbellSeries`. [M]/[F]

**Read the motor bounds from here; never hardcode them** — they differ per model.
Note `liveviewTracking`/`autoTracking` in `smartDetect` are **PTZ features, not
sensor classes**. [M]

A controller `features` block observed to work:
`{"mic": true, "aec": [], "videoMode": ["default"], "motionDetect": ["enhanced"]}`. [M]

---

## 4. Settings messages

### `ChangeVideoSettings`

```json
{
  "audio": { "bitRate":32000, "channels":1, "description":"audio track",
             "enableTemporalNoiseShaping":false, "enabled":true, "mode":0,
             "quality":0, "sampleRate":11025, "type":"aac", "volume":0 },
  "firmwarePath": "/lib/firmware/",
  "video": {
    "enableHrd":false, "hdrMode":0, "lowDelay":false,
    "videoMode":"default", "vinFps":30,
    "video1": { … }, "video2": { … }, "video3": { … }, "mjpg": { … }
  }
}
```

Per-H.264-track object:

```json
{
  "M":1, "N":30,
  "avSerializer": {
    "destinations": ["tcp://<controller-ip>:7550/s0"],
    "parameters": { "audioId":1000, "streamName":"video1",
                    "suppressAudio":false, "suppressVideo":null, "videoId":null },
    "type": "extendedFlv"
  },
  "bitRateCbrAvg":1400000, "bitRateVbrMax":2800000, "bitRateVbrMin":48000,
  "description":"Hi quality video track", "enabled":true, "fps":15,
  "gopModel":0, "height":1512, "horizontalFlip":false, "isCbr":false,
  "maxFps":30, "minClientAdaptiveBitRate":0, "minMotionAdaptiveBitRate":0,
  "nMultiplier":6, "name":"video1", "sourceId":0, "streamId":1,
  "streamOrdinal":0, "type":"h264",
  "validBitrateRangeMax":2800000, "validBitrateRangeMin":32000,
  "validFpsValues":[1,2,3,4,5,6,8,9,10,12,15,16,18,20,24,25,30],
  "verticalFlip":false, "width":2688
}
```

**Track geometry (G5 PTZ, measured):**

| name | sourceId | streamId | ordinal | W×H | cbrAvg | vbrMax | vbrMin | validMax |
|---|---|---|---|---|---|---|---|---|
| `video1` | 0 | 1 | 0 | 2688×1512 | 1 400 000 | 2 800 000 | 48 000 | 2 800 000 |
| `video2` | 1 | 2 | 1 | 1280×720 | 500 000 | 1 200 000 | 48 000 | 1 500 000 |
| `video3` | 2 | 4 | 2 | 640×360 | 300 000 | 200 000 | 48 000 | 750 000 |
| `mjpg` | 3 | 8 | 3 | 1280×720 | 500 000 | 500 000 | `null` | 6 000 000 |

MJPEG track: `type:"mjpg"`, `avSerializer.type:"mjpg"`,
`destinations:["tcp://<ctrl>:7551/mjpg"]`,
`parameters:{audioId:1000, enableTimestampsOverlapAvoidance:false,
streamName:"mjpg", suppressAudio:true, suppressVideo:false, videoId:1001}`,
`fps:5`, `maxFps:5`, `quality:80`. [M]

**Critical shape rules** [M]:

1. **`avSerializer.parameters` must carry the full five-key shape** —
   `audioId`, `streamName`, `suppressAudio`, `suppressVideo`, `videoId` (the four
   non-name keys may be `null`). With only `{streamName}` the camera **silently
   drops the entire message**: no ack, no stream. Diagnose by **checking for the
   ack**, not by staring at the video path.
2. Destinations are `scheme://host:port/path` and the netloc is parsed — the host
   must be the real destination address.
3. **Empty `destinations: []` disarms a track** and the camera closes that push.
4. **The camera echoes `ChangeVideoSettings` back** with its own version — that
   echo is the authoritative record of what it accepted, kept, dropped, or
   overrode. Use it.

**Fields the camera silently overrides or drops** [M]:
- `audio.volume` → **force-reset to 0 in the echo, every time.**
- A per-track `audio{}` sub-object → **dropped**; audio config is device-level only.
- `nMultiplier` coerced; `vinFps` recomputed from `videoMode`. [F]

### `ChangeIspSettings`

~60 imaging fields. A known-good payload:

```
aeMode:"auto", aeTargetPercent:50, aggressiveAntiFlicker:0, brightness:50,
contrast:50, criticalTmpOfProtect:40, darkAreaCompensateLevel:0, denoise:50,
enable3dnr:1, enableMicroTmpProtect:1, enablePauseMotion:0, flip:0,
focusMode:"ztrig", focusPosition:0, forceFilterIrSwitchEvents:0, hue:50,
icrLightSensorNightThd:0, icrSensitivity:0, irLedLevel:215, irLedMode:"auto",
irOnSts{Brightness,Contrast,Denoise,Hue,Saturation,Sharpness,Wdr}:0,
irOnVal{Brightness,Contrast,Denoise,Hue,Saturation,Sharpness}:50, irOnValWdr:1,
mirror:0, queryIrLedStatus:0, saturation:50, sharpness:50,
touchFocusX:1001, touchFocusY:1001, wdr:1, zoomPosition:0
```
[M]

**Send the full object with overrides applied** — a partial push risks the camera
re-defaulting unset fields. `aeTargetPercent` is the exposure target;
`zoomPosition` here is also the **zoom command** (zoom belongs to the ISP daemon,
not the gimbal). [M]

### `ChangeOsdSettings`

```json
{ "_1": {"enableDate":1,"enableLogo":0,"enableReportdStatsLevel":0,
         "enableStreamerStatsLevel":0,"tag":"<name>"},
  "_2": {...}, "_3": {...}, "_4": {...},
  "enableOverlay":1, "logoScale":50, "overlayColorId":0,
  "textScale":50, "useCustomLogo":0 }
```

`_1`.._4` are four identically-shaped overlay slots. **`enableLogo:0` removes the
vendor watermark**; `enableOverlay` must stay `1` or nothing renders. [M]

### `ChangeDeviceSettings` / `NetworkStatus` / `ChangeSoundLedSettings`

```json
ChangeDeviceSettings: {"name":"<camera name>", "timezone":"PST8PDT,M3.2.0,M11.1.0"}

NetworkStatus: {"connectionState":2, "connectionStateDescription":"CONNECTED",
  "defaultInterface":"eth0", "dhcpLeasetime":86400, "dnsServer":"8.8.8.8 4.2.2.2",
  "gateway":"…", "ipAddress":"…", "linkDuplex":1, "linkSpeedMbps":100,
  "mode":"dhcp", "networkMask":"255.255.255.0"}

ChangeSoundLedSettings: {"ledFaceAlwaysOnWhenManaged":1, "ledFaceEnabled":1,
  "speakerEnabled":1, "speakerVolume":100, "systemSoundsEnabled":1,
  "userLedBlinkPeriodMs":0, "userLedColorFg":"blue", "userLedOnNoff":1}
```
[M] — `NetworkStatus` is the controller *informing* the camera of its network.

### `ChangeSmartDetectSettings` — the object-detection on-switch

```json
{ "enableSmartDetect": ["person","vehicle","animal","alrmSmoke","alrmCmonx","alrmBabyCry"],
  "zones": { "0": { "coord":[0,0, 1000,0, 1000,1000, 0,1000],
                    "objectTypes":["person","vehicle","animal"],
                    "sensitivity":50, "crosslineDirection":"none",
                    "triggerLight":false } } }
```

The detector **runs regardless of adoption** but ships with `enableSmartDetect: []`
— no classes active, nothing ever fires. This message populates it. Key names are
parsed **verbatim**. `enableSmartDetect` takes **both** object and acoustic
`alrm*` classes; only object classes belong in a zone's `objectTypes` (an acoustic
alarm has no location). `coord` is an 8-value polygon in a 0..1000 frame.
**Acked, applied live, connection stays stable.** [M]

### `ChangeAudioEventsSettings`

Flat object, one key per class, **integers `0|1` — JSON booleans are silently
ignored**. Full schema in `unifi-camera-reference.md` §10. The daemon is
**boot-disabled**, so sending this over the control channel is hazardous — §6. [M]

### `ChangeNvrSettings` — untested, but its neighbours are revealing

Adjacent binary keys: `rtspEnabled`, `rtspPort`, `rtspUsername`, `rtspPassword`,
`rtspAuthEnabled`, `ntpClientHost`, `recordFulltime`, `recordMotionEvents`,
`recordSoundEvents`, `standaloneEnabled`, `connectionHost`,
`connectionSecurePort`, `secureTransfer`. **The leading candidate for enabling the
camera's own RTSP server and its NTP client.** [U]

---

## 5. Streaming

### Arming

A track streams **iff** its `destinations` is non-empty. Set to arm, `[]` to
disarm. The camera then opens a **plain TCP connection (no TLS)** and writes. [M]

**Snapshot pattern:** arm `video1` → decode one frame → push settings with empty
destinations. Each toggle costs **~10–12 s** of spin-up/tear-down, so a 15 s
interval yields ~2 frames/min, not 4. [M]

**Connection rotation:** the camera opens a fresh control WebSocket every few
seconds and several overlapping :7550 connections per track during re-arms. A
sender captured at arm time is on a dead socket by disarm time — **re-resolve the
live control connection at both ends.** [M]

### Multi-track demux — one port, by `streamName`

**The camera ignores per-track destination ports.** Pointing `video2`/`video3` at
:7552/:7553 dialled neither — it opened a fresh **:7550** connection per track.
The only discriminator is the **`streamName`** in each connection's `onMetaData`.
Demux by name, never by port; the `/sN` path suffix is cosmetic. [M]

> An independent implementation (`scrypted-unifi-direct`, on a G5 Turret Ultra
> fw 5.3.90) reports using per-track ports 17550–17552 successfully. Either this
> is model- or firmware-dependent, or one of the two findings is wrong.
> **Worth re-testing.**

`streamName` extraction: `onMetaData` is an AMF0 ECMA-array; keys are bare AMF0
strings (u16 BE length + UTF-8). Immediately after the literal bytes `streamName`
sits the value: type marker **`0x02`**, u16 length, then bytes. [M]

`onMetaData` advertises `name`, `hasAudio`, `suppressAudio`, `hasVideo`,
`suppressVideo`, `withTalkback`, `streamName` — and, even on a video-only push,
`hasAudio:true, audioChannels:1, audioFrequency:16000, audioBandwidth:64000`. [M]/[F]

### The extendedFlv container

Begins with ASCII `FLV` but is **not standard FLV**:

```
[0..8]    FLV header:  'F' 'L' 'V' 0x01 <flags> 00 00 00 09
                                        └─ flags = 0x07   (standard FLV: 0x05)
[9..12]   PreviousTagSize0 = 00 00 00 00

per tag:
  [0]       tag type   (8 audio, 9 video, 18 script)
  [1..3]    data size  (24-bit BE)
  [4..6]    timestamp  (24-bit BE)
  [7]       timestamp extended
  [8..10]   stream id  (always 0)
  [11..]    tag data
  ── end of standard 11-byte header + body ──
  [+4]      previous-tag-size
  [+16]     WALL-CLOCK TRAILER          ◀── the vendor extension
```

**20 bytes between tags where standard FLV has 4.** [M]

**Conversion to standard FLV** (verified byte-clean into `ffmpeg -f flv`):
1. Pass the 9-byte header through but **force byte 4 (flags) to `0x05`**.
2. Pass `PreviousTagSize0` unchanged.
3. Per tag: emit `tag_bytes` + a **freshly computed** 4-byte previous-tag-size
   (`= 11 + data_size`), then **skip the camera's 4-byte size and 16-byte trailer**.

Raw, `ffmpeg` fails with `Packet mismatch` — it reads the trailer as a bogus tag.

A deframer must buffer partial tags across chunk boundaries (a tag routinely
straddles two `recv()` calls) and needs `tag_len + 4 + 16` bytes before advancing. [M]

**Decodable prologue** — the minimum a fresh decoder needs to join:
`FLV header + PreviousTagSize0 + (script tag) + the AVC sequence header`. The AVC
sequence header is tag type `9` with `data[0] & 0x0f == 7` and `data[1] == 0`; a
real coded frame has `data[1] == 1`. Because the camera opens overlapping
connections per track, arbitrate per `streamName` (prefer the connection emitting
the most tags) and replay the cached prologue to late joiners. [M]

### MJPEG on :7551

Framing undocumented. What works: scan for JPEG **SOI `FF D8`** … **EOI `FF D9`**
and carve each complete span, discarding bytes in between. Bytes before the first
SOI are a header of unknown shape. [M]

### Audio in the push — closed

The camera **advertises an AAC track it never sends**: ~100–118 video tags and
**0 audio tags** over full 2 MB captures; `ffmpeg` reports `audio:0kiB`. Every
lever tried and failed — see `unifi-camera-reference.md` §8 for the full table and
the two-volumes root cause. [M]

---

## 6. Protocol hazards

| Hazard | Symptom |
|---|---|
| **Blasting the settings suite back-to-back** | Acks stop after the first seven; the socket resets ~6 s later and the burst re-fires — endless adoption loop (21 000+ handshakes observed). **Fix: ack-gate — release message N+1 only when N is acked.** [M] |
| **`ChangeAnalyticsSettings`** | **Resets the camera** — connection drops instantly, empty or full payload. A *different* daemon's handler. **Never send it.** [M] |
| **`ChangeAudioEventsSettings` while its daemon is down** | Tears down the control channel → ~7 s reset loop. The daemon is boot-disabled; the schema is correct but has no recipient. [M] |
| **Trailing fire-and-forget suite** | `ChangeAnalyticsSettings`, empty `ChangeSmartDetectSettings`, `UpdateUsernamePassword` are **never acked** and the connection resets ~2 s after the third. Adoption is functionally complete after the acked head — video already flows. [M] |
| **Waiting forever for an ack** | Those three are never acked even when processed. Use a ~3 s timeout to advance. [M] |
| **Controller sends no WebSocket ping** | Camera resets every ~6 s and re-runs adoption. Ping every ~2 s. [M] |
| **Losing the control channel** | Camera resets video destinations — the push dies with it. [F] |
| **`avSerializer.parameters` missing the 5-key shape** | Camera **silently drops** the whole message. Looks like an adoption bug. [M] |
| **Controller UUID drift** | `UUID mismatch` — no acks, no video, survives reboot. [M] |
| **Raw extendedFlv into a standard FLV decoder** | `Packet mismatch`. [M] |
| **Keying stream demux on port** | All H.264 lands on one port. Key on `streamName`. [M] |
| **Poll-after-move for PTZ position** | Reads stale/default; looks like a dead gimbal. Listen *before* you move. [M] |

> **Instrument-validation rule.** A silent sniffer proves nothing unless you have
> shown it isn't silent when it shouldn't be. Validate the instrument with a
> known-good operation in the same session — and validate the *stimulus* too. A
> well-instrumented test of a malformed message is still the wrong test.

---

## 7. Events

**The camera pushes events unprompted once a class is enabled. The controller
never polls.** [M]

| functionName | Payload | Edges |
|---|---|---|
| `EventAnalytics` | `{eventType:"motion", edgeType:…}` | **start / stop** [I] |
| `EventSmartDetect` | as above **+** `objectTypes:["person",…]` | **enter / leave** [M] |
| `EventSmartMotion` | same shape as SmartDetect | enter / leave [I] |
| `EventSmartAudio` | bare `alrm*` class tokens + `eventType`, `eventId`, `duration`, `levels`, `leveldB`, `enter`, `leave` | enter / leave [F] |
| `EventMotorState` | `{state:{position:{pan,tilt,zoom,focus}, activity, wallClockMs}}` | broadcast only while moving [M] |

**Active** edges: `start`, `enter`, `update`. **Cleared**: `stop`, `leave`, `end`.
A bare `Event*` with no `edgeType` is safest read as "happening now". [I]

Object class tokens: `person`, `vehicle`, `animal`, `face`, `package`,
`licensePlate`, `vehicleColor`, `vehicleType`, `faceMask`, `lineCrossing`. [F]

Acoustic tokens (event **output** vocabulary — deliberately different from the
`enableAlrm*` settings **input** keys): `alrmSmoke`, `alrmCmonx`, `alrmSiren`,
`alrmSpeak`, `alrmBabyCry`, `alrmBurglar`, `alrmCarHorn`, `alrmBark`,
`alrmGlassBreak`, `loudNoise`, `soundLoss`. [F]

Other families: `EventIspSceneStatus`, `EventIspSettingsChanged`,
`EventPoorNetwork`, `EventManagedConnStatusChanged`, `EventLensState`,
`EventWrongAttitude`, `EventFeatureFlagsUpdated`, `EventIavState*`,
`EventUpdateFirmwareStatus`, `EventCommandStatus`, `EventPtzControllStatus` *(sic)*. [F]

---

## 8. PTZ is not on this protocol

**Measured:** the *identical* `AbsolutePosition` payload moved the head **1310
motor units over SSH/IPC** and **0 over :7442**. `ubnt_avclient` contains the PTZ
handler strings, but its WebSocket dispatcher does not expose them — they are
IPC-only, and the camera hosts no inbound PTZ port. [M]

Full PTZ detail — the IPC transport, `AbsolutePosition`, the speed scale, the
listen-before-you-move rule, and the unsolved `PtzWebsockHandler` second channel —
is in `unifi-camera-reference.md` §9.

**Caveat on negative results:** `ubnt_avclient` registers **zero `onRequest_*`
symbols** (its routing is table-driven through a `CatchAllHandlerRejector`), so
"no handler symbol found" is **not** evidence that avclient cannot handle a verb. [F]

---

## 9. Open questions

1. **What bootstraps the PTZ WebSocket?** `PtzWebsockHandler` dials a URI given at
   runtime; the delivery mechanism is unfound. Best lead: watch a genuine
   controller adopt the camera.
2. **How does a real controller sync the camera clock?** The `timeSync` reply is
   not applied. NTP is the hypothesis.
3. **What sets `av.audio.volume`?** Controller-managed state absent from every
   on-camera store; it gates whether an audio track is muxed at all.
4. **`ChangeNvrSettings`** — could plausibly enable the camera's own RTSP server
   and NTP client. Entirely untested, and the highest-value unexplored lead.
5. **Per-track push ports** — reconcile the single-port/`streamName` finding here
   against `scrypted-unifi-direct`'s 17550–17552.
6. **`ubnt_avclient`'s dispatch table** is unmapped; every "avclient has no handler
   for X" conclusion is unproven.
7. **`RelativePosition` / `ContinuousMove` payload shapes** — guessed shapes acked
   but produced no movement ("looks fine, does nothing").
8. **MJPEG inter-frame framing** — only the JPEG markers are understood.
9. **`EventAnalytics` / `EventSmartAudio` exact payloads** — inferred from firmware
   strings and a camera-side reference implementation, not captured live.
10. **UDP :10001 discovery payload** — hex-dumped, never decoded. (An HN commenter
    describes it as a proprietary TLV protocol.)
11. **Talkback** (`ChangeTalkbackSettings`) — entirely unmapped.
12. **`GetRequest`** — believed a snapshot-upload trigger; shape unconfirmed.
