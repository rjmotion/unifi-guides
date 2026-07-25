# flock — project brief

A family of tools that impersonate one side of the UniFi Protect camera↔console
relationship, so each side can be studied, tested, and used independently of the
other.

---

## The names

```
flock/
├── cuckoo/      fake console — real cameras adopt TO it
└── lyrebird/    fake camera  — a real console adopts IT
```

| Scope | Name | Why |
|---|---|---|
| **Umbrella** | **flock** | The plain collective noun — the group the birds belong to. Unpretentious, obvious in a directory listing, and it scales: any future impersonator just joins the flock. |
| **Console emulator** — a fake NVR that real cameras adopt to | **cuckoo** | Already built and earned. A cuckoo is a *brood parasite*: it plants its egg in another bird's nest and the host raises it as its own — impersonate one half of a trusted pairing until the real device accepts you. Also nods to Clifford Stoll's *The Cuckoo's Egg*. |
| **Camera emulator** — a fake camera that a real console adopts | **lyrebird** | The finest mimic in nature: it reproduces chainsaws, car alarms and — literally — **camera shutters**. A project that imitates a camera well enough to fool its controller has no better namesake. |

Each bird carries a distinct idea rather than variations on one: **cuckoo**
parasitises, **lyrebird** mimics — and the **flock** is simply what they belong to.

---

## Prior art

### Camera side — solved, and mature

**`unifi-cam-proxy`** — https://github.com/keshavdv/unifi-cam-proxy
MIT · ~1.9k stars · ~261 forks · active · Docker + PyPI + Discord.
Docs: https://unifi-cam-proxy.com

Presents a **fake camera to a real UniFi Protect console**, so third-party
(RTSP) cameras appear natively in the Protect UI and app. Supports live
streaming, full-time recording, motion detection (camera-dependent), and smart
detections when paired with Frigate.

Adoption requires two things:
- **A client certificate** — either lifted from a real UniFi camera
  (`scp ubnt@<cam>:/var/etc/persistent/server.pem client.pem`) or self-generated
  with OpenSSL. Passed as `--cert`.
- **An adoption token** from the console
  (`https://{NVR}/proxy/protect/api/cameras/manage-payload`), valid **60 minutes**
  and invalidated by a service restart.
- `--mac` sets a unique MAC per instance (follows MAC-randomisation rules).

Video is supplied by ffmpeg from an RTSP source. It is a *fake camera*, so it
never has to make real camera hardware do anything — which is why it sidesteps
the whole class of problems that come with driving genuine hardware.

**`unifi-proxy`** — https://github.com/Gamer08YT/unifi-proxy
A second, smaller implementation of the same idea. Less established; worth a look
for a different take on the adoption path.

### Controller side — one peer, two weeks old

**`scrypted-unifi-direct`** — https://github.com/xuio/scrypted-unifi-direct
TypeScript · 0 stars · 20 commits · **first commit 2026-07-11**, active.

*"Use UniFi Protect cameras in Scrypted directly — no NVR, Cloud Key, or
Console."* It **emulates the Protect controller** so real cameras adopt to it —
the same problem, independently solved, and it arrived only days ago. Its
architecture converged remarkably: TLS management server on **:7442**, cameras
aimed via **`controller.addr`**, commands the camera to push **extendedFlv** over
TCP and strips Ubiquiti's trailers to recover standard FLV. It ships its own
in-process RTSP server (FLV→RTP in JS, **no ffmpeg/mediamtx**).

Two things it does **not** do, and one disagreement worth resolving:
- **No PTZ.** Its README doesn't mention gimbal control at all.
- **No ONVIF.** It outputs into Scrypted, not as a standard ONVIF device.
- **It uses per-track push ports 17550–17552**, which contradicts the measured
  finding here that the G5 PTZ ignores per-track ports and pushes every H.264
  track to a **single** port, demultiplexed by FLV `streamName`. Either this is
  model-dependent or one of us is wrong. **Reconcile this early** — it's cheap to
  test and it changes the receiver design.

Read it closely. It is the only independent implementation of this side.

### Console *firmware* emulation — still doesn't exist

Distinct from the above, and still open: actually **booting UniFi OS**. What
exists:

- **A partial QEMU boot, documented once.**
  https://emulatedbox.wordpress.com/2024/12/12/emulating-ubiquity-dream-machine-firmware-booting-into-user-space/
  (Dec 2024) — UniFi OS / Dream Machine Pro **4.0.20**, extracted with
  `binwalk -Me` (ARM64 kernel + squashfs + cpio initramfs), booted under
  `qemu-system-aarch64 -machine virt -cpu max -m 4G`. The stock kernel
  (4.19.152-ui-alpine) produced no console output, so the author cross-compiled a
  6.9.5 aarch64 kernel. It reaches a BusyBox/initramfs shell and then **fails: no
  `/dev/mtdblock5`**, so the real root filesystem never mounts and UniFi OS
  userspace never starts. **No repo, no scripts.** This is a lab note, not a tool.
- **`unifi-utilities/unifios-utilities`** — https://github.com/unifi-utilities/unifios-utilities
  ~4.4k stars, very active. Runs custom scripts at boot **on real UniFi OS
  hardware** (UDM/UDM-Pro/UDR/UNVR, UniFi OS 4.x+, via systemd). Not emulation.
- **`fabianishere/udm-kernel-tools`** — custom kernels on real UDM hardware. Not
  emulation.
- **`hjdhjd/unifi-protect`** — https://github.com/hjdhjd/unifi-protect
  A near-complete **client** for the Protect API (TypeScript). Talks *to* a real
  console; does not pretend to be one.

### Device side (fake AP/switch/gateway adopting to a real controller) — mature

Not our direction, but the same trick, and the best-developed corner of the field:

- **`jamesbraid/unifi-emu`** — https://github.com/jamesbraid/unifi-emu
  Go, brand new, active. *"Fake UniFi devices that speak the inform protocol and
  get adopted by a real controller."* **182 device models**, CBC→AES-GCM cipher
  negotiation, full adoption to CONNECTED, emulated firmware-upgrade reboots,
  both ClassicClient and UOSClient paths. Strict rule: devices enter only through
  the real inform lifecycle, never DB seeding. The most complete emulator found
  anywhere in the ecosystem.
- **`amd989/unifi-gateway`** (UGW3, Python, maintained) and its archived ancestor
  **`stephanlascar/unifi-gateway`**, whose `unifi_protocol.py` is the widely-copied
  reference implementation.
- **Inform protocol docs** (no code, but the reference material):
  `jeffreykog/unifi-inform-protocol` (118★, docs only),
  `fxkr/unifi-protocol-reverse-engineering`.

### What genuinely does not exist

- **No open-source UniFi Network Application replacement.** Nothing provisions
  real APs. `imperian-systems/unifi-controller` is a 2021 Laravel inform-receiver
  stub, backend-only, dead.
- **No standalone mock UniFi API server**, for Network or Protect. `aiounifi` uses
  `aioresponses`; `uiprotect` uses `AsyncMock` + recorded JSON. Anyone wanting a
  black-box HTTP/WS fake has to build it. (`uiprotect`'s
  `unifi-protect generate-sample-data` anonymising recorder is good prior art.)
- **Nobody does PTZ.** Not `scrypted-unifi-direct`, not `unifi-cam-proxy`, nobody.
  Gimbal control against a real UniFi PTZ camera — the SSH/IPC `AbsolutePosition`
  path and the `EventMotorState` feedback loop — has **no public counterpart**.
- **Nobody re-exports adopted UniFi cameras as standard ONVIF.** Every ONVIF
  project in this space points the other way (feeding third-party cameras *into*
  Protect: `daniela-hase/onvif-server`, `BigTonyTones/Tonys-Onvf-RTSP-Server`).
- **No UniFi OS firmware emulator** (see above — one partial QEMU attempt).

---

## Where the white space is

Keep three different meanings of "emulator" apart; they need entirely different work:

1. **Protocol-level, controller side** — speak the camera-facing protocol well
   enough that a real camera adopts and streams. **Solved twice now**: cuckoo, and
   `scrypted-unifi-direct` as of two weeks ago.
2. **Protocol-level, device side** — solved well (`unifi-cam-proxy` for cameras,
   `unifi-emu` for APs/switches).
3. **Firmware-level** — boot the actual firmware in QEMU. Attempted once,
   partially, and stalled. **Genuinely unsolved**, and the harder, more
   interesting problem.

**The defensible ground for this project is PTZ and ONVIF re-export** — both
completely unclaimed — plus firmware-level emulation if you want the hard problem.

---

## Suggested shape

```
flock/
├── README.md                  what this family is, and the two directions
├── docs/
│   ├── camera-reference.md    everything known about the camera as a device
│   └── protocol-reference.md  the camera↔console wire protocol
├── cuckoo/                    fake console (exists)
├── lyrebird/                  fake camera (new; study unifi-cam-proxy first)
└── firmware/                  unpacking, emulation experiments, notes
```

Start `lyrebird` by reading `unifi-cam-proxy` rather than from scratch — it has
solved certificate handling and the adoption-token dance already, and its MIT
licence makes that reuse clean. The value to add is on the *protocol* side, where
the knowledge in `docs/` goes far deeper than any published implementation.

---

## Sources

**Controller-side emulation**
- https://github.com/xuio/scrypted-unifi-direct — the one peer; read it closely

**Camera-side emulation**
- https://github.com/keshavdv/unifi-cam-proxy · https://unifi-cam-proxy.com
- https://github.com/Gamer08YT/unifi-proxy

**Device-side emulation (AP/switch/gateway → real controller)**
- https://github.com/jamesbraid/unifi-emu — 182 models, the most complete
- https://github.com/amd989/unifi-gateway · https://github.com/stephanlascar/unifi-gateway
- https://github.com/trueserve/openUF

**Inform-protocol reference (docs, no emulator)**
- https://github.com/jeffreykog/unifi-inform-protocol
- https://github.com/fxkr/unifi-protocol-reverse-engineering
- https://github.com/dmke/inform-inspect · https://github.com/mcrute/go-inform

**Firmware-level emulation**
- https://emulatedbox.wordpress.com/2024/12/12/emulating-ubiquity-dream-machine-firmware-booting-into-user-space/
- https://github.com/unifi-utilities/unifios-utilities · https://github.com/fabianishere/udm-kernel-tools

**API clients / test fixtures (prior art, not emulators)**
- https://github.com/hjdhjd/unifi-protect · https://github.com/uilibs/uiprotect
  (note its `unifi-protect generate-sample-data` anonymising recorder)
- https://github.com/Kane610/aiounifi

**Discussion**
- https://news.ycombinator.com/item?id=47310152 (camera adoption; UDP 10001 TLV)
- https://news.ycombinator.com/item?id=47308278 (inform protocol)
