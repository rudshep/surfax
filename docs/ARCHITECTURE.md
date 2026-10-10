# SURFAX Architecture

> **Status:** design phase, nothing here is built yet.
> **Audience:** anyone who wants to understand *why* SURFAX looks the way it does, and where they can help.

This document has two jobs:

1. Record the **thought process** that led to the design, so nobody has to re-argue settled questions.
2. Define the **MVP** small enough that a handful of volunteers can finish it.

If you disagree with a decision, open an issue titled `ADR: <topic>` and point at the section. Decisions are meant to be challenged with evidence, not defended out of loyalty.

---

## 1. The idea in one paragraph

**SURFAX** (Surface Facsimile) turns almost any device with a working screen into an extra display for a computer, a VM, or an app. The experience goal is *"plug in a monitor and it shows up in Display Settings"*, but with a thirteen-year-old tablet or an €8 ESP32 panel standing in for the monitor.

The pixels are not the hard part (VNC solved that decades ago). The hard part is the **experience**: install, discovery, pairing, identity, resolution, touch and recovery. SURFAX adds a small protocol for that, **RCSDP** (Remote Control Surface Discovery Protocol), and reuses VNC/RFB for the pixels.

A surface is more than a screen. Think **screen, keyboard, mouse and controller**: a synth module with knobs, a robot controller with joysticks, an audio player with transport buttons and album art. RCSDP therefore describes the **display surfaces *and* the control surfaces** of a device (buttons, knobs, sliders, joysticks, LEDs), and the host turns them into standard virtual devices or a simple event API. See [RCSDP-CONTROLS.md](RCSDP-CONTROLS.md).

## 2. Vocabulary

| Term | Meaning |
|---|---|
| **Surface** | Any thing that shows pixels: ESP32 panel, old tablet, browser tab, a CRT behind a bridge. |
| **Host** | The machine running SURFAX software that produces content. |
| **Agent** | The SURFAX program on the host. Discovers surfaces, pairs, starts sessions. |
| **Source** | Where a session's pixels come from: a desktop region, a VM console, a headless browser, etc. |
| **Session** | One source delivered to one surface. |
| **Head** | A *real* OS-level virtual monitor (post-MVP). |
| **Bridge** | Hardware that receives a stream and outputs analog or digital video (post-MVP). |
| **Profile** | A small data file describing a device model: resolution, pin map, colour order, quirks. |
| **Endpoint** | A group of related capability on a surface: display, pointer, keyboard, or controller. |
| **Control** | One thing a person can touch, turn, push or see: a button, knob, slider, joystick, LED. |
| **Role** | What a control is *for*, e.g. `media.play_pause` or `gamepad.left_stick`. Gives meaning to raw controls. |
| **Binding** | The host-side projection of a surface's controls: a virtual gamepad, a JSON event stream, MIDI, OSC, etc. |

## 3. How we got here (the thought process)

### 3.1 Three layers, so each can be replaced

Everything splits into **sources/heads** (host side), **protocol** (the wire), and **surfaces** (device side). Only the surface-specific and OS-specific bits should ever be platform code. Everything in the middle stays portable.

### 3.2 Can we make real second monitors on every OS?

Yes in principle, but the maturity varies a lot. These are the options as we understand them (verify before depending on any of them):

| OS | Mechanism | Difficulty |
|---|---|---|
| Windows | Indirect Display Driver (IddCx), user-mode | Good API. Driver signing is the real cost. |
| Linux | EVDI (DKMS) or compositor virtual outputs | Works, but DKMS and Secure Boot hurt. Compositors differ. |
| macOS | Private `CGVirtualDisplay` API | Fragile across OS updates. Riskiest platform. |

**Conclusion:** real heads are worth building, but they are the *slowest and riskiest* part. They must not block the MVP.

### 3.3 Where does the novelty actually live?

Not in a codec. Not even in virtual displays (EVDI, IddCx samples and several commercial second-screen apps already exist). The gap is:

* Surfaces that are **too old for the apps** that existing second-screen products need.
* A **plug-and-play experience** (auto-discovery, auto-config, a stable identity, one-click setup) for dumb surfaces.
* **One protocol** that treats a $10 ESP32, an iPad 1 and a CRT bridge as equals.

### 3.4 Should we build on VNC (RFB)?

We looked at what RFB already gives us and where it falls short.

| RFB gives us | RFB lacks |
|---|---|
| Client chooses pixel format (RGB565 on an ESP32) | The client never describes *itself* (native size, DPI, touch) |
| Client ranks the encodings it can decode | No "this is a display looking for content" discovery |
| Pull-based updates, so a slow client can't be flooded | Weak classic authentication (old DES challenge) |
| Rectangles + CopyRect (cheap scrolling) | The efficient encodings lean on zlib, heavy for tiny chips |
| Supported by QEMU/Proxmox, x11vnc, wayvnc, TigerVNC and every debugging tool | Assumes the *viewer* is the interesting end |

Three options were considered:

| Option | Verdict |
|---|---|
| **A.** Pure RFB with pseudo-encoding extensions | Compatible, but our novel parts get buried in awkward corners. |
| **B.** Brand-new protocol | Clean, but throws away the ecosystem and every existing tool. |
| **C.** **Split planes**: new small control plane + RFB subset as the pixel plane | **Chosen.** |

### 3.5 How heavy is an ESP32 player?

Our working assumptions for a stock Cheap Yellow Display (ESP32-2432S028, ~520 KB SRAM, no PSRAM, 320×240 ILI9341, resistive touch):

* The player logic itself is small. The Wi-Fi/TCP stack dominates RAM and flash.
* It must **stream**: receive a rectangle, decode into a small scratch buffer, push to the panel by DMA. No full framebuffer (that alone would be ~150 KB).
* Cheap encodings (Raw RGB565, Hextile) fit easily. JPEG tiles are the likely sweet spot for photo/video-like content. zlib-based encodings are possible but expensive. H.264 is out of scope.
* The bottleneck is probably Wi-Fi throughput and SPI, not CPU.

> ⚠️ These are **estimates, not measurements**. Milestone M0 exists to replace them with real numbers.

### 3.6 What makes setup feel like plugging in a monitor?

Ideas we ranked by impact. The first five are in the MVP:

1. Browser-based flashing for ESP32 (no toolchain).
2. Wi-Fi credentials pushed to the device (no typing on a tiny screen).
3. **Surface advertises, host connects.** The surface never needs the host's address.
4. **Persistent identity + a number on screen.** The surface shows "Surface 7". That number is the handle in the picker and in Proxmox.
5. **Profiles by model ID** to tame the "CYD variant zoo" (see 3.7).
6. *(later)* USB bootstrap: plug in once, host flashes and configures.
7. *(later)* Synthesised EDID with stable serial so the OS remembers the monitor layout.
8. *(later)* Host-managed OTA firmware updates.

### 3.7 Why profiles matter (evidence)

Public CYD tutorials repeatedly hit the same wall: swapped touch axes, mirrored coordinates, and red/blue colour order that differ between board revisions and have to be fixed by hand in config. Every user rediscovers the same fixes. A community-maintained **profile database** turns each fix into reusable knowledge. This is both the best onboarding path for contributors and a long-term moat.

### 3.8 What do the big streaming protocols teach us?

We looked at RDP, Sunshine/Moonlight and RustDesk, not as products to copy but for how they handle **frame buffers and input devices**. Full notes are in [PRIOR_ART.md](PRIOR_ART.md). The lessons that changed our design:

| Lesson | Effect on SURFAX |
|---|---|
| RDP separates concerns into **virtual channels** | Display, input and device control are separate logical channels. |
| Moonlight uses **short-PIN pairing**, then trust | Matches our on-screen number and pairing code. |
| Moonlight sends **absolute pointer + reference size** | Touch maps cleanly to any host resolution. |
| RDP touch handling never drops down/up transitions | We classify events into **edge / delta / level** with different reliability. |
| Sunshine sends **rumble, LED and battery** feedback | A first-class host→surface output path. |
| RustDesk derives **stable IDs from a key pair** | Surface identity can be cryptographic later. |
| Every one of them needs a real video decoder or heavy stack | Confirms: RFB subset for the pixels, no hardware codec on the critical path. |

Net result: **VNC remains the pixel plane**, and we borrow ideas, not code. (RustDesk is AGPL-3.0 and Sunshine is believed to be GPL-3.0, so copying code would constrain SURFAX's own licence.)

### 3.9 Surfaces are more than screens

The original idea was "an extra monitor." The use cases (synth modules, robot controllers, audio players) revealed that the surface's **knobs, buttons, sticks and LEDs** matter as much as its pixels. The question became: how does a device *describe* its controls so the host can use them without per-device code?

Prior art gave us three pieces (see PRIOR_ART.md §4):

* **USB HID** shows self-describing hardware is solved, but raw HID gives numbers without meaning and needs quirk tables.
* **MIDI 2.0's MIDI-CI** adds *profiles* (named bundles of expected behaviour) and two-way negotiation with fallback.
* **OSCQuery** shows the right discovery shape: advertise a tiny record over mDNS, then serve the detailed tree on request.

We combine them: advertise small, describe on request, cache by hash, add **roles** and **profiles** for meaning, and project to **standard host devices**. Details in [RCSDP-CONTROLS.md](RCSDP-CONTROLS.md).

### 3.10 Why Android 4 is in the MVP

Old Android tablets are *the* archetypal drawer device, and they test our hardest claim (an old OS is still a useful surface). They also come with real obstacles: no WebSocket in old WebViews, TLS disabled by default before API 20, no modern discovery guarantees, and a security posture that demands network isolation. We decided to face those early. Details are in [ANDROID4.md](ANDROID4.md).

## 4. The architecture

```
┌──────────────────────── HOST ────────────────────────┐
│                                                       │
│   Sources                     SURFAX Agent            │
│  ┌────────────────┐         ┌───────────────────┐     │
│  │ Desktop region │──┐      │ RCSDP: discover,  │     │
│  │ Proxmox VM     │──┼─RFB─▶│ pair, assign #,   │     │
│  │ Headless browser│─┤      │ start sessions    │     │
│  │ (future) Heads │──┘      └─────────┬─────────┘     │
│  └────────────────┘                   │               │
└───────────────────────────────────────┼───────────────┘
                                        │  RCSDP (control)  +  RFB (pixels)
              ┌─────────────────────────┼──────────────────────┐
              ▼                         ▼                      ▼
        ESP32 CYD               Old tablet browser        Bridge (future)
        native player           Tier 0 page               → CRT / VGA / RF
```

### 4.1 Three channels

| Channel | Carries | Built on |
|---|---|---|
| **Control (RCSDP)** | Discovery, pairing, identity, descriptors (screens *and* controls), identify, attach/detach, power, status, firmware info | mDNS/DNS-SD + a tiny message protocol over a stream |
| **Pixel** | Frames and rectangles, plus pointer events from touching a display | A deliberately small **subset of RFB** |
| **Input / outputs** | Control events (buttons, encoders, sticks, keys) surface→host, and LED/haptic/label outputs host→surface | RCSDP messages, classified as edge / delta / level |

The control and input channels are the new things. The pixel channel is the proven thing. Any normal VNC viewer can attach to a session for debugging, which is a big deal for a community project. Touch on a display rides the RFB pointer path so any VNC server works; richer controls use the RCSDP input channel. We avoid sending the same input twice.

### 4.2 Sources are pluggable

The agent doesn't care where pixels come from. A source just has to expose an RFB endpoint at the surface's resolution. This one idea covers a lot of ground:

| Source | How it works | Phase |
|---|---|---|
| **Desktop region** | Capture a rectangle of an extended desktop and serve it over VNC | MVP |
| **Proxmox VM console** | QEMU already exposes VNC. Agent just points the surface at it. | MVP |
| **Headless browser** | Render any web UI (Home Assistant dashboard, synth UI, robot dashboard) at the surface's size | MVP |
| **Real OS head** | EVDI / IddCx / CGVirtualDisplay | Post-MVP |

### 4.3 RCSDP v0 (sketch)

*Names and fields are proposals. Finalised in milestone M1.*

**Discovery.** Surface advertises `_rcsdp._tcp` via mDNS/DNS-SD with TXT records:

| Key | Meaning |
|---|---|
| `v` | RCSDP version |
| `id` | Stable surface ID (from hardware) |
| `name` | Friendly name |
| `model` | Profile ID (e.g. `esp32-2432s028`) |
| `w`, `h` | Native resolution |
| `fmt` | Pixel formats (e.g. `rgb565`) |
| `enc` | Encodings the player can decode, best first |
| `in` | Input capabilities (`touch`, `keys`, …) |
| `fw` | Player firmware version |
| `disp` | Number of display endpoints |
| `ctl` | Number of controls (summary only) |
| `cd` | Hash of the full control descriptor, so hosts can cache it |

**Control messages** (over TCP, format decided in M1; JSON is the leading candidate for readability):

| Message | Direction | Purpose |
|---|---|---|
| `HELLO` | surface → host | Full descriptor |
| `PAIR` | both | One-time pairing using a code shown on the surface |
| `ASSIGN` | host → surface | "You are Surface 7" |
| `IDENTIFY` | host → surface | Flash the number so the user can tell which is which |
| `ATTACH` | host → surface | "Connect as an RFB viewer to host:port with this pixel format/encoding" |
| `DETACH` | host → surface | End session, go back to idle screen |
| `POWER` | host → surface | Backlight on/off/dim |
| `STATUS` | surface → host | Heartbeat, link quality, errors |

**Why "host tells surface where to connect"?** The surface only ever makes an *outgoing* viewer connection, which is the simplest thing for a tiny device. RFB's reverse-connection mode remains an option.

**Numbers.** On first contact the surface shows a short **pairing code**. Once paired, the host assigns a persistent **Surface Number**, which is displayed whenever the surface is idle or identified. Proxmox and the UI both refer to it (e.g. tag `surfax-7`).

### 4.4 Surfaces are screens *and* controls

RCSDP describes a surface as a set of **endpoints**: display, pointer, keyboard and controller. Controls (buttons, encoders, knobs, sliders, joysticks, hats, LEDs, haptics, per-control labels) carry a kind, range, optional **role** and layout hints.

| Mechanism | Purpose |
|---|---|
| **Small advert, detailed descriptor on request** | mDNS carries only counts and a hash; the full control tree is fetched once and cached. |
| **Roles and profiles** | Give raw controls meaning (`gamepad`, `media-remote`, ...) so the host can bind them to standard devices. Community-maintained data files. |
| **Edge / delta / level classes** | Presses never drop, encoder steps are summed, analog levels are latest-wins. |
| **Fail-safe on disconnect** | The host releases every pressed control and re-centres sticks. No stuck inputs. |
| **Host bindings** | Linux virtual HID (uinput) and a JSON event stream in the MVP; OSC, MIDI, ROS and Home Assistant later. |

The full model, examples (synth module, robot controller, audio player) and open questions are in [RCSDP-CONTROLS.md](RCSDP-CONTROLS.md).

### 4.5 Security stance

SURFAX targets old devices that cannot do modern TLS. We will not pretend otherwise.

* Pairing creates a shared secret. The control plane authenticates with it.
* The pixel plane uses classic VNC authentication per session, which is weak.
* **SURFAX is for trusted LANs and VLANs.** Never expose it to the internet.
* Modern surfaces (browsers, ESP32-S3 class) may get stronger options later.

## 5. MVP

### 5.1 Scope

**In:**

* Linux host agent with discovery, pairing and sessions
* **ESP32 CYD** player firmware (the reference low-end surface)
* **Android 4 surface** (native app, API 16 to 19; browser fallback for older). See [ANDROID4.md](ANDROID4.md)
* **Control surface descriptors** (buttons, encoders, sticks, LEDs) with a small DIY reference controller, bound on Linux to a virtual HID device and a JSON event stream. See [RCSDP-CONTROLS.md](RCSDP-CONTROLS.md)
* RCSDP v0 and a written spec
* Sources: desktop region, Proxmox VM console, headless browser
* Touch → pointer events using the RFB pointer path
* Profile format + the first few CYD profiles

**Out (explicitly, for now):**

* Real OS heads (EVDI, IddCx, macOS)
* Windows and macOS agents
* The hardware bridge
* EDID synthesis
* Strong encryption

### 5.2 Milestones

| # | Milestone | "Done" looks like |
|---|---|---|
| **M0** | **Measure.** Run an existing ESP32 VNC viewer on a real CYD against a stock VNC server. | We have real numbers: frame rate and free heap for Raw/Hextile. We decide if RFB-subset is good enough or whether a private JPEG-tile encoding is needed. |
| **M1** | **Discovery.** CYD advertises itself, the agent lists it. | `surfax list` shows the CYD with its resolution within seconds of power-up. |
| **M2** | **Pairing + identity.** Pairing code on screen, persistent number. | "Surface 7" survives a reboot of both ends. |
| **M3** | **Sessions.** Agent starts a source at the surface's size and tells it to attach. | The CYD shows a live desktop region. |
| **M4** | **Touch.** Taps become pointer events. | You can click a button on the CYD and it works on the host. |
| **M5** | **Controls.** Control descriptor, fetch-and-cache by hash, Linux virtual HID and JSON event stream, fail-safe release on disconnect. Reference DIY controller with 2 encoders, 4 buttons, a thumb joystick and a few LEDs. | Turn a knob and a host app sees it; pull the Wi-Fi and nothing stays stuck. |
| **M6** | **Android 4 surface.** Native app plus agent-served APK and QR/short-URL install. | An Android 4.1+ tablet is installed, paired, shows Surface N and a live session. |
| **M7** | **Sources.** Proxmox VM console and headless-browser (Home Assistant dashboard) sources. | A VM tagged `surfax-7` appears on Surface 7 at boot. |
| **M8** | **Tier 0 surface.** Plain browser page (MJPEG/polled) for the oldest tablets. | An Android 4.0 or iPad 1 shows a session by opening a URL. |

Android experiments A0 to A5 (see ANDROID4.md §5) can start in parallel with M1.

After the MVP: ESPHome external component and a Home Assistant add-on, Windows/macOS agents, real heads, the bridge.

### 5.3 Success criteria

1. Flash a CYD, power it, and see it in the host's list **without editing a config file**.
2. Two surfaces at once, each with a stable number.
3. Unplug the router and plug it back in: surfaces recover on their own.
4. A new contributor can add a device profile **without writing C++**.
5. An old Android tablet goes from "in the drawer" to "Surface N" using **only a URL and the tablet**.
6. A surface with knobs and buttons shows up on the host as a **standard virtual gamepad or a JSON event stream** with no per-device code.
7. Dropping the link never leaves a button stuck or a joystick off-centre.

### 5.4 Assumptions we need to test

| Assumption | Test | Milestone |
|---|---|---|
| A CYD can run a streaming RFB-subset player with tens of KB of RAM | Measure heap on real hardware | M0 |
| Raw + Hextile give acceptable UI performance over Wi-Fi | Measure frame rate | M0 |
| JPEG tiles are needed for video-like content | Compare against Hextile | M0 |
| mDNS works reliably on a typical home Wi-Fi | Test on several routers | M1 |
| An old browser can show a polled-JPEG or MJPEG surface | Test on real old devices (WebSocket is absent in Android WebViews before 4.4, and old Safari support is uncertain) | M8 |
| A tiny dependency-free Java app builds and installs on Android 4.1 to 4.4 | Hello-world APK on three real devices | A0 |
| NsdManager is reliable enough for discovery on Android 4 | Compare against a UDP beacon fallback on real Wi-Fi | A1 |
| Edge/delta/level classes keep controls responsive over Wi-Fi without stuck inputs | Reference controller over a congested network | M5 |
| A JSON control descriptor is small enough for an ESP32 to serve | Measure flash and RAM cost | M5 |

## 6. Proposed repository layout

```
surfax/
├── docs/
│   ├── ARCHITECTURE.md        (this file)
│   ├── PRIOR_ART.md           what we learned from RDP, Moonlight, RustDesk, HID, MIDI-CI
│   ├── RCSDP-CONTROLS.md      screens + controls model
│   ├── ANDROID4.md            Android 4 surface plan and obstacles
│   └── rcsdp/                 protocol spec, versioned
├── agent/                     host agent (Linux first)
│   └── bindings/              uinput, JSON events (later: OSC, MIDI, ROS)
├── surfaces/
│   ├── esp32-cyd/             player firmware
│   ├── esp32-controller/      reference DIY control surface
│   ├── android4/              native app (tiny, dependency-free)
│   └── browser/               Tier 0 web surface
├── sources/
│   ├── desktop-region/
│   ├── proxmox/
│   └── headless-browser/
├── profiles/                  device profiles (data files, easiest PR!)
├── roles/                     control roles and profiles registry (data files, also easy PRs)
├── tools/                     web flasher, test utilities
└── CONTRIBUTING.md
```

## 7. How to contribute

Pick your own level. You don't need to understand the whole system.

| If you have… | You can… |
|---|---|
| **A drawer of old devices** | Test and add a **profile** for your device. This is the most valuable contribution. |
| **An ESP32 and a CYD variant** | Run the M0 measurements and share the numbers. |
| **Networking knowledge** | Review the RCSDP draft, test mDNS on odd routers. |
| **Linux / systems skills** | Help with the agent and sources. |
| **Web skills** | Build the Tier 0 browser surface. |
| **Home Assistant / ESPHome experience** | Prototype the HA dashboard source and ESPHome component. |
| **Technical writing** | Setup guides, device guides, diagrams. |
| **Driver experience (IddCx, EVDI, macOS)** | Feasibility notes for the post-MVP heads. |
| **An Android 4 tablet** | Run the A0 to A5 experiments and file a device test report (see ANDROID4.md §6). |
| **Android/Java skills** | Help build the tiny dependency-free Android 4 app. |
| **MIDI, HID or OSC experience** | Review the control model, propose roles, build the MIDI/OSC bindings. |
| **Synth, robotics or audio-player hobbyists** | Build a reference controller and tell us which controls and roles you'd actually need. |

Ground rules:

* Small pull requests. One idea each.
* Measurements beat opinions. Include numbers and device models.
* Every claim about a device should say which exact model and OS version it was tested on.
* Be kind. Someone's dusty tablet is somebody's first project.

Suggested licences (to be confirmed): **Apache-2.0** for code, **CC-BY-4.0** for the protocol spec.

## 8. Use cases

The three that drive the design:

1. **Home automation touch panels** (ESPHome + Home Assistant)
2. **Modular synthesizer** display and control modules
3. **Robotics controllers**

Details and honest trade-offs are in the [README](../README.md). The common thread is that the UI is rendered on a host and the cheap device only shows it and reports its controls (touch, knobs, buttons, sticks). A fourth shape, an **audio player** with transport buttons, a volume control and album art, falls out of the same design.

### When SURFAX is *not* the right tool

* If you want a **self-contained** ESP32 panel, ESPHome's built-in display and LVGL support is already good and runs without any host. SURFAX is for people who want host-rendered content.
* For **hard real-time** or **safety-critical** control (audio-rate paths, robot e-stops), never route that through SURFAX. It carries pictures and input hints, not safety.
* If your device is recent enough to run a current second-screen app, use that.

## 9. Prior art and reading

Full notes live in **[PRIOR_ART.md](PRIOR_ART.md)**. Summary:

* **VNC / RFB**: the pixel channel. Existing ESP32 VNC viewer libraries are the starting point for M0 (we haven't audited them yet).
* **Microsoft RDP, Sunshine/Moonlight, RustDesk**: studied for frame buffer handling, input and pairing. Ideas adopted: channels per concern, short-code pairing, absolute pointer with reference size, edge/delta/level input classes, host→device feedback. No code copied.
* **USB HID, MIDI 2.0 (MIDI-CI), OSCQuery**: studied for describing controls. We combine "advertise small, describe on request" (OSCQuery), profiles with fallback (MIDI-CI), and self-describing controls (HID) plus roles for meaning.
* **EVDI**, **Windows IddCx sample driver**, **CGVirtualDisplay** based tools: the future heads.
* **Commercial second-screen apps** (Duet Display, Spacedesk, Apple Sidecar and similar): good UX benchmarks, but they target current OS versions, which is exactly the gap SURFAX fills.
* **ESPHome + LVGL** on the CYD: the incumbent for standalone panels.
* **Chromecast / AirPlay** discovery UX: the experience bar we are aiming at.

## 10. Evidence behind the use cases (and its limits)

What we found when checking for demand (searched October 2026):

* The CYD is widely used as a Home Assistant panel; there are blog posts, a Hackaday feature (September 2026), and 3D-printable enclosures.
* Several tutorials report the same CYD configuration gotchas (touch axis swap, mirroring, colour order).
* Many "repurpose your old tablet" articles list *second monitor*, *smart-home dashboard*, *camera viewer* and *system monitor* as the standard ideas, and they mostly depend on apps that old OS versions can't install.

**Limits of this evidence:** it is blog posts and listicles. It shows people *want* to repurpose old hardware. It does **not** prove they want SURFAX specifically. We should validate with real users early (see M0/M1 community calls).

**Not found / not verified:** any existing project that matches the full SURFAX combination. We did not do an exhaustive code search. If you know of one, please open an issue; we'd rather collaborate than duplicate.

## 11. Open questions

1. Final control-plane message format (JSON vs a compact binary).
2. Private JPEG-tile encoding vs Tight-subset (decided by M0).
3. Which headless browser to use for the HA source, and how to keep it light.
4. How the agent handles multiple hosts discovering the same surface.
5. Naming hygiene: a Python materials-science package called *Surfaxe* exists (different spelling and domain). We found no existing project named RCSDP. A trademark/package-name check is still worth doing before wide release.
6. Wire format for descriptors and events (JSON vs CBOR vs compact binary), and whether the input channel needs a UDP path for low-jitter analog streams.
7. How many control roles ship in v0, and who curates the registry.
8. Android 4: lowest `minSdk` we can really support (16 vs 14) and how to fall back on 4.0.x.
9. Windows and macOS virtual HID for control bindings: current driver and API options are unresearched.
10. Licence hygiene: confirm the licences of any library we might link or study (RustDesk is AGPL-3.0; Sunshine and moonlight-common-c are believed GPL-3.0).
