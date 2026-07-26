# unifi-guides

**Reverse-engineering notes on how UniFi Protect cameras talk to their console —
and how to make either side talk to something you wrote instead.**

UniFi cameras cannot be used without a Ubiquiti console. They have no standalone
mode, no ONVIF, and they don't serve RTSP: they dial *outward* to a controller
over a modified FLV transport, and pan/tilt/zoom is driven entirely by that
controller. So without one you have a camera you cannot configure and a motorised
head you cannot steer.

These documents are what's been established about that relationship, in enough
detail to reimplement either end of it. They're the research base for
[flock](https://github.com/rjmotion?tab=repositories), which builds tools that
impersonate one side or the other.

**Device of record:** UVC G5 PTZ, firmware 5.3.95 / middleware v4.70.91, observed
against real UniFi Protect 7.1.77.

---

## Where to start

| If you want to… | Read |
|---|---|
| Know how any of this was found out | **[`techniques.md`](techniques.md)** |
| See at a glance what's confirmed, and how well | **[`conformance-matrix.md`](conformance-matrix.md)** |
| See what a real controller actually sends | **[`real-controller-observations.md`](real-controller-observations.md)** |
| Implement the wire protocol | [`protocol-reference.md`](protocol-reference.md) |
| Work on the camera as a device — SSH, daemons, PTZ | [`unifi-camera-reference.md`](unifi-camera-reference.md) |
| Run Protect without Ubiquiti hardware | [`unifi-protect-off-hardware-guide.md`](unifi-protect-off-hardware-guide.md) |
| Understand the landscape and prior art | [`flock-project-brief.md`](flock-project-brief.md) |

**Read `techniques.md` before attempting a new measurement.** The method is
usually already there, and it will save you the two or three hours it cost to
find the first time.

---

## The documents

### [`techniques.md`](techniques.md) — how to get at any of this

Twelve techniques, ordered by time saved. The most useful: **the controller will
log its own plaintext if you ask it to.** `:7442` is TLS, but the daemon serving
it has payload tracing built in and disabled by default — no interception or
debugger needed, and neither would work easily anyway.

Also: reassembling hex log dumps into JSON, parsing pcaps without tshark, walking
Ubiquiti's `extendedFlv` container, reading config schemas out of stripped
binaries, testing a disk-management daemon without risking your disks, and which
log file holds what.

### [`real-controller-observations.md`](real-controller-observations.md) — what a real controller does

The experiment the other documents kept asking for: run genuine Protect
off-hardware, let it adopt a real camera, and watch. It answers the project's
largest open question — **what makes the camera's dormant PTZ WebSocket client
dial out** — and overturns several previously documented conclusions, including
that in-band audio was impossible and that PTZ wasn't on this protocol at all.

§9 is built from 263 plaintext JSON envelopes and a packet capture. Where §9 and
the earlier sections disagree, §9 wins.

### [`protocol-reference.md`](protocol-reference.md) — the wire protocol

Transport, ports, TLS, the WebSocket envelope, the adoption handshake and what
actually gates it, the settings vocabulary, the `extendedFlv` container, events,
and the protocol hazards — including the messages that will reset a camera.

### [`unifi-camera-reference.md`](unifi-camera-reference.md) — the camera as a device

OS, daemons, the IPC bus, SSH, the media pipeline, PTZ, detection, firmware
format, and its failure modes. **Read §11 first if you're debugging** — nearly
every hour lost on this platform was lost to something in there.

### [`unifi-protect-off-hardware-guide.md`](unifi-protect-off-hardware-guide.md) — Protect without Ubiquiti hardware

The containerised approach, the two upstream projects, firmware extraction, and
why booting the firmware under QEMU is a dead end.

### [`flock-project-brief.md`](flock-project-brief.md) — the landscape

Prior art survey, the three different things people mean by "emulator", and where
the genuinely unclaimed ground is.

---

## Confidence tags — please respect them

Every claim carries one:

| Tag | Meaning |
|---|---|
| `[MEASURED]` | Observed on real hardware, or on the wire |
| `[MEASURED-LOG]` | Read from the controller's own log statements — strong, but one step from the transport |
| `[FIRMWARE]` | Read from binaries or strings |
| `[INFERRED]` | Reasoned from the above |
| `[UNVERIFIED]` | Present in the vocabulary, never exercised |

A large share of the time lost on this platform went to treating an inference as a
measurement. The tags exist so you don't repeat that — and so do the correction
logs, which record claims that turned out wrong rather than quietly deleting them.

---

## Related

| Repo | What it is |
|---|---|
| [`unifi-unvr-emu`](https://github.com/rjmotion/unifi-unvr-emu) | Runs real UniFi Protect in a container on non-Ubiquiti arm64 hardware. The instrument most of this was measured with |
| [`unifi-library`](https://github.com/rjmotion/unifi-library) | Fetches every project these documents cite, for offline reading and cross-repo grep |

## Licence and standing

[CC BY 4.0](LICENSE) — share and adapt with attribution.

Independent research. UniFi, UniFi Protect and UNVR are products of Ubiquiti Inc.;
this project is not affiliated with, endorsed by, or supported by them, and
contains none of their software.
