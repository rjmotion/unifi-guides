# Running UniFi Protect Off Ubiquiti Hardware — Project Guide & Reference

Background material for a new project based on **`unifi-protect-arm64`** (containerised UniFi Protect).
Compiled 25 July 2026. Every link below was fetched and verified on that date unless marked otherwise.

---

## 1. Problem statement

UniFi Protect is closed-source and officially runs only on Ubiquiti console hardware: a Cloud Gateway,
CloudKey Gen2+, UDM/UDM-Pro/UDM-SE, or UNVR. Ubiquiti's self-hosted product (**UniFi OS Server**) hosts
the *Network* application only — Protect is explicitly excluded. This is a commercial decision, not a
technical one: Protect carries no licence fee, so the hardware is the product.

Consequences for anyone with a UniFi camera and no console:

- **Cameras cannot be configured standalone.** G4/G5-generation cameras have no supported standalone mode.
- **No ONVIF.** Generic PTZ/NVR tooling does not work.
- **Streams are push, not pull.** Cameras connect *outward* to a controller over a modified FLV
  transport rather than serving RTSP. You cannot simply point ffmpeg at the camera.
- **PTZ is controller-mediated.** Pan/tilt/zoom is driven through Protect's API; without a controller
  you have a motorised camera you cannot steer.

Therefore: to use a UniFi camera fully, you need something that behaves like a Protect controller.

---

## 2. The three approach families

| # | Approach | Viability | Verdict |
|---|----------|-----------|---------|
| **A** | Run the real Protect binaries off-hardware in a container | **Working, in production use by others** | ✅ Chosen basis |
| **B** | Emulate the console hardware under QEMU and boot stock firmware | Single abandoned experiment, never reached userspace | ❌ Dead end |
| **C** | Write a "fake NVR" that speaks the camera adoption/stream protocol | Protocol partly documented, nobody has built it | ⚠️ Unbuilt; good fallback reference |

Approach A is the basis for the new project. B and C are documented below because they contain
protocol and firmware detail that A does not, and because C is the fallback if A proves
architecture-bound.

---

## 3. Primary basis — `unifi-protect-arm64`

**Repo:** <https://github.com/markdegrootnl/unifi-protect-arm64>
**Status:** Archived by owner **23 March 2026** — read-only. 145 stars, 24 forks, 13 open issues.
**Licence/legal posture:** Ships no Ubiquiti software. States it is unaffiliated with Ubiquiti and uses
only packages freely available on the internet. You supply the firmware yourself.

### How it works

1. Download UNVR firmware from Ubiquiti's public download page.
2. Extract the Debian packages from it.
3. Drop them into `put-deb-files-here/` and `put-version-file-here/`.
4. `docker build -t markdegroot/unifi-protect-arm64 .`
5. Run the container; Protect's web UI is served on `https://localhost/`.

Extraction is documented at
<https://github.com/markdegrootnl/unifi-protect-arm64/blob/master/doc/Extract_deb_files_from_firmware.md>

### Run invocation

```bash
docker run -d --name unifi-protect \
    --privileged \
    --tmpfs /run \
    --tmpfs /run/lock \
    --tmpfs /tmp \
    -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
    -v /storage/srv:/srv \
    -v /storage/data:/data \
    -v /storage/persistent:/persistent \
    --network host \
    -e STORAGE_DISK=/dev/sda1 \
    markdegroot/unifi-protect-arm64
```

### Hard requirements

| Requirement | Detail |
|---|---|
| **Architecture** | **ARM64 only.** The debs are UNVR aarch64 packages. |
| Storage | Protect refuses to start with less than **100 GB** free. |
| Privileges | `--privileged`, host networking, cgroup mount. |
| Init system | systemd runs as PID 1 inside the container. |
| Kernel | May need `systemd.unified_cgroup_hierarchy=0` (cgroup v1) — see [moby/moby#42275](https://github.com/moby/moby/issues/42275). |

### Known gotchas

- **"Device Updating" hang.** After initial setup a popup with a blue loading bar can stick. Fix:
  `systemctl restart unifi-core` inside the container, or restart the container. First boot only.
- **Cloud remote access needs `enp0s2`.** Remote access via Ubiquiti's cloud fails unless the host's
  primary interface is literally named `enp0s2`; the README gives a systemd `.link` file to rename it.
  *Irrelevant for a local-only deployment — skip this.*
- **systemd-in-Docker errors** (`Failed to allocate manager object: Read-only file system`) — the
  cgroup v1 kernel parameter above.

### The better starting point — `unifi-unvr-arm64`

**Repo:** <https://github.com/snowsnoot/unifi-unvr-arm64> (fork of the above; ~40 stars)
**Docker Hub image:** `snowsnoot/unifi-unvr:latest`

Two material improvements over upstream:

1. **Automated firmware extraction.** `scripts/fwextract.sh {firmware_url}` replaces the manual
   binwalk-and-copy dance, then `docker build`.
2. **Protect version currency — the important one.** The UNVR firmware bundle on Ubiquiti's site
   contains an *outdated* Protect deb. The README documents pulling a newer one:

   ```bash
   curl https://fw-update.ubnt.com/api/firmware-latest | jq
   curl -o firmware/unifi_protect_<version>.deb {updated_protect_deb_url}
   rm firmware/unifi_protect_<old_version>.deb
   ```

   Swap the deb before `docker build`. **This is the lever that decides whether newer cameras
   (e.g. G5 PTZ) are recognised at all**, and it means "archived repo" matters less than usual —
   the container is mostly scaffolding around packages fetched fresh.

It also adds a `--tmpfs /var/opt/unifi-protect/tmp` mount, plus a `docker-compose.yml` and a systemd
unit for running it as a service.

### Open questions for the new project

- **Host architecture.** ARM64 native is the happy path. On x86_64 you are looking at qemu-user
  emulation of a continuous video-processing workload — expect this to be unusable, and validate
  early rather than late.
- **Minimum Protect version for target cameras.** Determine which Protect release first supports the
  camera model, then confirm a matching deb is reachable via `firmware-latest`.
- **PTZ through the container.** Should work, since this *is* real Protect — but unverified.
- **Both upstreams are archived/stale.** Assume no maintainer; budget for owning the Dockerfile.

---

## 4. Approach B — firmware emulation under QEMU (dead end, documented)

**Source:** "emulating ubiquity dream machine firmware – booting into user space", EmulatedBox
(Stephan M.), 12 December 2024 —
<https://emulatedbox.wordpress.com/2024/12/12/emulating-ubiquity-dream-machine-firmware-booting-into-user-space/>

A single blog post. **No repo, no scripts.** A lab note, not a tool.

What was achieved:

1. UniFi OS Dream Machine Pro **4.0.20** firmware `.bin` downloaded from Ubiquiti's public page.
2. `binwalk -Me` extraction, yielding an ARM64 kernel Image, a squashfs, and a cpio initramfs.
3. Boot attempt via `qemu-system-aarch64 -machine virt` with `-kernel` and `-initrd`.

Where it stopped, and why it is a dead end:

- The stock kernel produced **no console output** at all; `console=ttyAMA0` and `console=ttyS0` both
  silent. Author's hypothesis: kernel not built with console device support.
- Workaround was to cross-compile a **6.9.5 aarch64** kernel, which boots and reaches init.
- **Blocker 1:** init waits for `/dev/mtdblock5`, times out, drops to a BusyBox 1.30.1 ash prompt.
  The real rootfs never mounts.
- **Blocker 2 (compounding):** the initrd expects modules under `/lib/modules/4.19.152-ui-alpine`.
  Substituting a 6.9.5 kernel breaks every `modprobe`. Fixing mtdblock5 alone is insufficient — a
  4.19.152 rebuild would also be required.
- **Blocker 3:** `ps aux` at the prompt shows no startup scripts ran. The UniFi userspace binaries
  aren't in the initrd at all — they're in the squashfs. "Booting into user space" in the title means
  the *initramfs* userspace, not UniFi OS.

The author hedged in his own opening ("maybe? probably?"), stated next steps as matching the 4.19.152
kernel and passing mtdblock5 via QEMU, and **never wrote a part two** — his blog moved on to CUDA,
Hashicorp Vault, then a multi-part EDR series, last posting August 2025.

**Why keep it on file:** the extraction procedure and the partition-layout clue (`mtdblock5` ≈ where
the squashfs rootfs lives) are useful if you ever need to pull binaries out of *console* firmware
rather than UNVR firmware.

### Related hardware research

- **Rick Mark, "Ubiquity UniFi Security and Boot-chain Analysis"** (July 2022) —
  <https://blog.rickmark.me/untitled-3/>
  Boot chain, EL2/EL3 usage, KVM presence, and the `/sbin/ssh-proxy` root-promotion oddity on UDM-Pro.
  Useful for understanding what the console hardware actually does at boot.
- **udm-utilities issue #131** (Feb 2021) —
  <https://github.com/boostchicken-dev/udm-utilities/issues/131>
  Early "has anyone made a QEMU image of UDM-Pro?" thread. Answer was no.

---

## 5. Approach C — the fake NVR (unbuilt; protocol reference)

Nobody has shipped a controller emulator that adopts a *real* UniFi camera and captures its stream.
Confirmed still open as of a Hacker News thread dated **10 March 2026**
(<https://news.ycombinator.com/item?id=47310152>), where the state of the art was described as SSHing
into the camera and hand-editing configs that don't survive reboot.

**But the protocol is documented — by projects doing the mirror image.** `unifi-cam-proxy` and its
forks impersonate a *camera* so third-party hardware can join Protect. Their reverse-engineering of
the camera↔controller wire format is directly reusable.

### Camera-side emulation projects

| Project | Language | Notes |
|---|---|---|
| **[keshavdv/unifi-cam-proxy](https://github.com/keshavdv/unifi-cam-proxy)** | Python | The original and most mature. Live streaming, full-time recording, motion detection, Frigate smart detections. Docs: <https://unifi-cam-proxy.com>. PyPI: `unifi-cam-proxy`. |
| **[NorthernMan54/unifi-cam-proxy-redalert](https://github.com/NorthernMan54/unifi-cam-proxy-redalert)** | Python | Fork-of-a-fork, 0 stars, no releases. **Value is the README, not the code** — best available written spec of the protocol. Stages 4–5 (the 7550 channel) are marked work-in-progress and not implemented. |
| **[Gamer08YT/unifi-proxy](https://github.com/Gamer08YT/unifi-proxy)** | Java | Documents the QR-code adoption-token flow (`/proxy/protect/api/cameras/qr`), 60-minute token expiry, JKS certificate handling, and controller-side log paths (`/srv/unifi-protect/logs/cameras.avclient.log`). |

### Protocol summary (from the redalert README)

Ubiquiti's own docs confirm adoption requires **TCP 7444, 7550, 7442 and UDP 10001**
(<https://help.ui.com/hc/en-us/articles/360012622613-UniFi-Device-Adoption>).

Five-stage handshake:

| Stage | Direction | Port | Transport | Purpose |
|---|---|---|---|---|
| 1 | Controller → Camera | 10001 | UDP | Discovery broadcast; camera replies with identity TLVs (MAC, firmware, feature bits) |
| 2 | Controller → Camera | 443 | HTTPS | `POST /api/1.2/manage` — adoption. Payload carries token, controller hostnames, credentials |
| 3 | Camera → Controller | 7442 | WSS | AVClient handshake: token auth, "hello" feature negotiation, baseline settings (video, ISP, motion) |
| 4 | Camera ↔ Controller | 7550 | WSS | Command/event bus |
| 5 | Camera ↔ Controller | 7550 | WSS | uPFLV video/audio/metadata stream |

**Stream format (uPFLV):** a UniFi magic prefix
`DE 19 16 15 47 17 DE 19 16 75 50` precedes a standard FLV header. Then AMF0 script tags
(`0x12`): `onMetaData` (bandwidth, fps, resolution, `extendedFormat`, random per-session
`streamName`), `onMpma` (per-module bitrate stats), `onClockSync` (`streamClock`, `streamClockBase`,
`wallClock`). Video tags (`0x09`) carry AVC sequence headers then raw H.264 NALUs with no extra
framing. Audio (`0x08`, AAC) is optional and appears only when the driver advertises `hasAudio`.
Motion/smart-detect events arrive as further `0x12` script tags interleaved with frames.

**Downlink (controller → camera):** lightweight binary frames prefixed `DE 19 16 75 50`, carrying
stream start/stop, heartbeats, and clock-sync. Loss of heartbeats should tear down the session.
⚠️ This direction is *described from a packet capture, not implemented* in any public project.

**Not documented anywhere:** PTZ commands. Camera-emulation projects had no reason to work out what a
pan/tilt/zoom instruction looks like on the wire — their drivers are static sources. The redalert
README lists PTZ only as a future aspiration.

**Caveat:** Ubiquiti changes the adoption flow between releases. See
<https://github.com/keshavdv/unifi-cam-proxy/discussions/351> — the "add camera" button was removed
and the `manage-payload` token URL began returning `Invalid token`. Any fake-NVR project inherits this
maintenance treadmill. (Note this churn is *controller*-side, so a project that **is** the controller
is less exposed than one pretending to be a camera.)

---

## 6. Controller-API client libraries (mature; assume you own a console)

These do not help you obtain a controller — but once the container from §3 is running, they are the
tooling you drive it with, and they are excellent.

| Project | Language | Notes |
|---|---|---|
| **[hjdhjd/unifi-protect](https://github.com/hjdhjd/unifi-protect)** | TypeScript | The reference implementation. First complete open-source implementation of the realtime update API, and the only complete implementation of the livestream API — direct H.264/HEVC/AV1 datastream access, not the controller's RTSP URLs. ESM-only, Node 22.18+. npm: `unifi-protect`. |
| **[hjdhjd/homebridge-unifi-protect](https://github.com/hjdhjd/homebridge-unifi-protect)** | TypeScript | HomeKit integration built on the above; pioneered the realtime-events reverse engineering that most other ecosystems then adopted. |
| **[Home Assistant `unifiprotect`](https://www.home-assistant.io/integrations/unifiprotect/)** | Python | Backed by the `uiprotect` library. Media source for clips and event thumbnails. |
| **[seaside1/unifiprotect](https://github.com/seaside1/unifiprotect)** | Java | openHAB binding. UniFi OS only. Documents the anonymous-snapshot trick for sub-10-second snapshot polling. |
| **[jeremycole/unifi_protect](https://github.com/jeremycole/unifi_protect)** | Ruby | CLI-first: list/describe cameras, pull snapshots, export recorded video by time range. Good for scripting. |

---

## 7. Standalone-camera hacks (fallback if the container path fails)

- **"Using UniFi Cameras Without UniFi Protect"**, Andrés Alejos —
  <https://dev.to/acalejos/using-unifi-cameras-without-unifi-protect-4k79>
  Adopts a G3 Flex into the legacy **UniFi Video** software (default creds `root`/`ubnt`, web app on
  `https://127.0.0.1:7443`) purely to enable RTSP, then feeds Frigate. Works for G3-era cameras.
- **[mx-shift/docker-unifi-video](https://github.com/mx-shift/docker-unifi-video)** — the legacy UniFi
  Video controller in a container. Predecessor product to Protect; ports 7080/7443/6666/7442/7445–7447.
  Only useful for older cameras.
- **Camera-local API.** On a camera flipped to standalone, session-cookie login plus `rest.cgi` and
  `/snap.jpeg` snapshot polling is usable (snapshot-mode Frigate). Undocumented outside forum lore;
  worth probing for motor control before assuming PTZ needs the full protocol.

---

## 8. Correction log

Claims previously made in conversation that turned out to be **wrong**, recorded so they don't get
repeated:

| Claim | Reality |
|---|---|
| "Protect cannot be self-hosted; blog posts claiming otherwise are wrong or extrapolating." | Ubiquiti's *official* position, yes. But the community containerised it years ago by extracting the debs. §3. |
| "There's no credible cracked/extracted Protect container floating around." | `unifi-protect-arm64` is exactly that: 145 stars, published Docker Hub image, working. |
| `unifi-cam-proxy-redalert` presented as having newly solved the downlink protocol. | It *documents* the downlink from a capture; its own roadmap marks stages 4–5 as work-in-progress. Documentation, not implementation. |

---

## 9. Reference index — all links

**Off-hardware Protect (approach A)**
- <https://github.com/markdegrootnl/unifi-protect-arm64> — primary basis (archived 2026-03-23)
- <https://github.com/markdegrootnl/unifi-protect-arm64/blob/master/doc/Extract_deb_files_from_firmware.md> — extraction procedure
- <https://github.com/snowsnoot/unifi-unvr-arm64> — fork with `fwextract.sh` + deb-swap
- <https://fw-update.ubnt.com/api/firmware-latest> — current firmware/deb index (JSON)
- <https://www.ui.com/download/releases/firmware> — Ubiquiti firmware downloads
- <https://github.com/moby/moby/issues/42275> — systemd-in-Docker cgroup issue

**Firmware emulation (approach B)**
- <https://emulatedbox.wordpress.com/2024/12/12/emulating-ubiquity-dream-machine-firmware-booting-into-user-space/> — the QEMU lab note
- <https://blog.rickmark.me/untitled-3/> — UniFi boot-chain security analysis
- <https://github.com/boostchicken-dev/udm-utilities/issues/131> — early QEMU-image thread

**Camera protocol / fake NVR (approach C)**
- <https://github.com/keshavdv/unifi-cam-proxy> — original camera emulator
- <https://unifi-cam-proxy.com> — its documentation site
- <https://github.com/NorthernMan54/unifi-cam-proxy-redalert> — best protocol write-up
- <https://github.com/Gamer08YT/unifi-proxy> — Java implementation, token/cert detail
- <https://github.com/keshavdv/unifi-cam-proxy/discussions/351> — adoption-flow churn
- <https://help.ui.com/hc/en-us/articles/360012622613-UniFi-Device-Adoption> — official port list
- <https://news.ycombinator.com/item?id=47310152> — March 2026 "has anyone done this" thread

**Controller API clients**
- <https://github.com/hjdhjd/unifi-protect>
- <https://github.com/hjdhjd/homebridge-unifi-protect>
- <https://www.home-assistant.io/integrations/unifiprotect/>
- <https://github.com/seaside1/unifiprotect>
- <https://github.com/jeremycole/unifi_protect>

**Standalone / legacy**
- <https://dev.to/acalejos/using-unifi-cameras-without-unifi-protect-4k79>
- <https://github.com/mx-shift/docker-unifi-video>

---

## 10. Suggested first steps for the new project

1. **Confirm host architecture.** ARM64 → proceed. x86_64 → spike qemu-user viability *before*
   anything else; if it fails, the project pivots to approach C.
2. **Fork `snowsnoot/unifi-unvr-arm64`**, not the upstream — it has the extraction script and the
   deb-swap procedure.
3. **Build with the newest Protect deb** from `firmware-latest`, not the one bundled in firmware.
4. **Validate camera adoption and PTZ early.** These are the two unproven assumptions.
5. **Modernise the container**: cgroup v2 if possible, drop `--privileged` if achievable, pin deb
   versions, add a build-time firmware-version manifest.
6. **Record what breaks between Protect releases.** That log is the project's real long-term value —
   nobody else is keeping one.
