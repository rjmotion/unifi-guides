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
| Source | `/srv/unifi-protect/logs/cameras.avclient.log`, `cameras.log` |

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

Tagged `[MEASURED-LOG]` throughout to keep this distinct from the `[MEASURED]`
(observed on the wire / on hardware) used elsewhere in `guides/`.

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
hello advertises `"audioCodecs":["aac","opus"]`, and the controller picks
**opus**. This is the strongest available candidate for the unsolved audio
problem. [MEASURED-LOG] — *not yet verified to actually produce audio tags; that
is the next experiment.*

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
