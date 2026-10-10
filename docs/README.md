<div align="center">

# SURFAX

### *Surface Facsimile*

**Don't recycle. Cycle.**

Turn the old phone, tablet or dev board in your drawer into a real extra screen.
Plug it in. Power it up. It shows up.

![status](https://img.shields.io/badge/status-design%20phase-orange)
![protocol](https://img.shields.io/badge/protocol-RCSDP%20v0%20draft-blue)
![contributions](https://img.shields.io/badge/contributions-very%20welcome-brightgreen)

</div>

---

## The drawer problem

Somewhere in your house there is a drawer. In it: an Android 4 tablet, an iPad 1, an ESP32 board with a lovely little touchscreen you bought for a project you never finished.

The screens still work perfectly. The *software* is what died. The app stores moved on, the OS is too old to run anything modern, and so the hardware sits there, a glowing rectangle waiting for a purpose.

**SURFAX gives it one.**

## What is SURFAX?

SURFAX makes almost any screen behave like a monitor that your computer, VM or smart home can send pictures to. And if the device has knobs, buttons, sliders or joysticks, those show up too, as a controller your software can use.

The goal is an experience like plugging in an HDMI monitor:

1. **Power on the old device.** It shows a number, say **Surface 7**.
2. **Your computer notices it.** No IP addresses, no config files.
3. **Pick it.** The screen lights up with whatever you sent it. Touch comes back as input.

The surface does almost no thinking. Your host does the rendering and the old device just shows pixels and reports taps. That's why a thirteen-year-old tablet and an €8 ESP32 board can both be first-class screens.

```
   ┌────────────┐         ┌──────────────────────────┐
   │  Your host │  RCSDP  │  Surface 7  (ESP32 panel)│
   │  PC / VM / │◀───────▶│  Surface 8  (old tablet) │
   │  Proxmox   │  + VNC  │  Surface 9  (CRT bridge) │
   └────────────┘         └──────────────────────────┘
     renders it             just shows it
```

## How it works (the short version)

SURFAX has two small parts:

* **RCSDP**, the *Remote Control Surface Discovery Protocol*. It handles the "plug in and it appears" part: discovery, pairing, a persistent number, resolution, identify, touch capability.
* **VNC (RFB)**, the proven protocol, for the actual pixels. Any normal VNC viewer can look at a SURFAX session, which makes debugging easy and keeps us honest.

Want the full reasoning? Read **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**. It explains every decision and what we haven't verified yet.

## Screen, keyboard, mouse and controller

A surface isn't just pixels. When a SURFAX device introduces itself, it can describe **everything it has**:

> *"I'm Surface 7. I have a 320×240 screen, touch, two push-encoders, four buttons, a thumb joystick and eight LEDs."*

The host then treats it like plugged-in hardware:

| The surface has... | The host sees... |
|---|---|
| A screen | An extra display session |
| Touch | A pointer |
| Keys | A keyboard |
| Knobs, sliders, joysticks, buttons | A **controller**: a standard virtual gamepad/device, or a simple event stream your software can read |
| LEDs, haptics, per-knob labels | Outputs your software can drive |

Three shapes this is designed for:

* **Synth module:** encoders and buttons plus a host-rendered scope or parameter page.
* **Robotics controller:** two joysticks and triggers plus a camera/telemetry screen.
* **Audio player:** play/pause/stop/next buttons and a volume knob, with album art on the display, and any player responds to the media keys.

Under the hood, controls are classified so they behave well over flaky Wi-Fi: button presses are never lost, encoder turns are summed, and analog positions always show the latest value. If the link drops, the host releases everything, so you never get a stuck button or a runaway joystick. Details in **[RCSDP-CONTROLS.md](RCSDP-CONTROLS.md)**.

> **Safety:** SURFAX is a display-and-control layer for trusted networks. Never route emergency stops or timing-critical signals through it.

## Use cases

### 🏠 1. Home automation touch panels (ESPHome + Home Assistant)

The Cheap Yellow Display is a popular €8-15 ESP32 board with a touchscreen, and it's a common Home Assistant panel project. ESPHome is excellent for this when you want a **standalone** panel with a hand-built layout.

SURFAX adds the other way to do it: render a full **Home Assistant dashboard on a host** and stream it to *any* screen, including that old tablet on the wall. Change the dashboard once and every surface updates, with no reflashing and no per-device YAML layouts.

*Planned:* an ESPHome component, so the panel still appears in Home Assistant as a normal ESPHome device.

### 🎛️ 2. Modular synthesizer displays and controls

Imagine a Eurorack-style module whose "screen" is a cheap panel showing a scope, a patch map or a sequencer grid that is rendered on a computer. Encoders, buttons and touch go back to the host, which maps them to MIDI, OSC or CV.

SURFAX treats the screen as a *display and control surface*, never the audio path. Timing-critical stuff stays in your synth, while the pretty and complicated stuff lives on a host.

### 🤖 3. Robotics controllers

A cheap ground-station screen for a robot: camera view, telemetry, a touch teleop pad. The same old tablet that can't run modern apps can show a host-rendered operator UI.

> **Safety note:** SURFAX carries pictures and input hints. Emergency stops and anything safety-critical must never depend on it.

### ➕ More ideas that look like real needs

We did some digging. These came up repeatedly in "what to do with an old tablet" write-ups and maker communities, and all of them are "send a picture to a dumb screen" problems:

| Use case | Why it fits |
|---|---|
| **Second monitor for your desk** | The classic reuse idea, but most apps for it need a recent OS on the tablet. |
| **Homelab / Proxmox wall display** | Show VM consoles and system stats on cheap panels. Built in to the roadmap. |
| **Camera wall** | One host builds a mosaic, any old screen shows it. |
| **Sim-racing / game dashboards** | Racing-game dashboards are already a known CYD project. |
| **Legacy CRT displays** | A planned bridge box could feed composite, VGA or RF to vintage screens. *(Our idea, not yet validated by users.)* |

> **A fair caveat:** the evidence is mainly blog posts and maker tutorials. It shows people *want* to repurpose old hardware. Tell us which of these you'd actually use.

## Supported surfaces

Everything is **planned**. Nothing is built yet. This table is the roadmap and the place where your device will appear.

| Surface | Tier | How you'd set it up | Status |
|---|---|---|---|
| ESP32 Cheap Yellow Display | Flash-and-go | Flash from the browser, power on | 🔬 first target |
| Old tablet with a browser (Android 4, iPad 1…) | Zero-install | Open a URL | 🧪 to be tested on real devices |
| **Android 4 tablet / phone** (4.1 to 4.4) | Tiny native app | Open a URL on the tablet, tap Install | 🔬 MVP, see [ANDROID4.md](ANDROID4.md) |
| Android 4.0.x | Browser page | Open a URL | 🧪 best effort |
| Windows CE and other kiosk shells | Tiny app | Install once | 💡 idea |
| DIY controller (ESP32 + knobs/buttons/joystick) | Flash-and-go | Flash from the browser | 🔬 MVP reference controller |
| Other ESP32 / ESP32-S3 displays | Flash-and-go | Community profiles | 💡 idea |
| CRT / composite / VGA / RF | Hardware bridge | Plug in the bridge box | 💡 idea |

## Hosts and sources

| Host / source | Status |
|---|---|
| Linux agent | 🔬 first target |
| Proxmox VM console on a surface | 🔬 MVP, tag a VM `surfax-7` and it appears |
| Headless browser (dashboards, web UIs) | 🔬 MVP |
| Real virtual monitors in the OS display settings (Windows, Linux, macOS) | 💡 after the MVP |

## Roadmap

| Step | What |
|---|---|
| **M0** | Measure a stock ESP32 VNC viewer on a real CYD, because numbers beat guesses |
| **M1** | A CYD announces itself and the host lists it |
| **M2** | Pairing code, then a persistent "Surface N" |
| **M3** | A live desktop region on the CYD |
| **M4** | Touch works |
| **M5** | Knobs, buttons and sticks appear as a virtual gamepad or event stream |
| **M6** | Android 4 tablet installed from a URL and showing a live session |
| **M7** | Proxmox VM and Home Assistant dashboard sources |
| **M8** | Zero-install browser surface for the very oldest tablets |

## What SURFAX is *not*

Honesty up front saves everyone time.

* **Not a way to play fast video on a tiny ESP32.** Expect UI-grade performance, and we'll publish real measurements.
* **Not secure over the internet.** Old devices can't do modern encryption. Use SURFAX on a trusted LAN or VLAN.
* **Not a replacement for ESPHome's built-in displays** if you want a self-contained panel with no host.
* **Not built yet.** This is a design-phase project that wants friends.

## Contribute

You don't need to be a protocol nerd. Honestly the best first contribution is the simplest:

> ### 🗄️ Open your drawer
> Tell us **what device you have**, its screen size, OS version and what happens when you try the browser surface. Each one becomes a **device profile** that helps the next person.

Other ways to help:

* **Have an ESP32 / CYD?** Help with the M0 measurements.
* **Know networking?** Review the RCSDP draft or test mDNS on your weird router.
* **Home Assistant / ESPHome person?** Prototype the dashboard source.
* **Good with words or diagrams?** Setup guides are gold.
* **Have an Android 4 tablet?** Run the first experiments and file a device report ([ANDROID4.md](ANDROID4.md)).
* **MIDI, HID or OSC person?** Review the control model and help define roles ([RCSDP-CONTROLS.md](RCSDP-CONTROLS.md)).
* **Know RDP, Moonlight, RustDesk or SPICE internals?** Check our notes in [PRIOR_ART.md](PRIOR_ART.md) and correct what we got wrong.
* **Have driver experience (Windows IddCx, Linux EVDI, macOS)?** Share feasibility notes for the future "real monitor" feature.

Start with **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**, then open an issue and say hello. Small pull requests, measurements over opinions, and kindness always.

## FAQ

**Why VNC and not a new protocol?**
VNC already handles pixels, rectangles, pull-based updates and picking encodings, and every tool speaks it. What it lacks is the "plug in and it appears" experience, so that's the only part RCSDP adds.

**Will it really show up in my OS display settings?**
Eventually, yes, via real virtual monitors. But those drivers are the hardest and riskiest part (signing on Windows, private APIs on macOS), so the MVP uses simpler sources first.

**Does something like this already exist?**
Pieces do: VNC, virtual display drivers, second-screen apps, ESPHome panels. We haven't found the combination of old-device support, auto-discovery and one protocol, but we haven't done an exhaustive search. If you know of a project, tell us. We'd rather join forces.

**How is this different from RustDesk, Moonlight or Microsoft RDP?**
Those are built to *control a remote computer* and expect a capable client with a real video decoder. SURFAX is built for *dumb, numbered, self-describing surfaces* (an ESP32, an Android 4 tablet) that you plug in like a monitor. We learned a lot from them, including channels per concern, PIN-style pairing and how they handle touch and controllers, but we borrow ideas, not code. See [PRIOR_ART.md](PRIOR_ART.md).

**Will my old Android 4 tablet really work?**
That's an MVP goal, with caveats: Android 4.1 to 4.4 gets a tiny native app, 4.0.x gets a browser fallback, and old devices have real security problems, so put them on an isolated network. Details and known obstacles in [ANDROID4.md](ANDROID4.md).

**Why "SURFAX"?**
**Surf**ace **Fax**simile, a faithful copy of your screen on a surface. The protocol is **RCSDP**: *Remote Control Surface Discovery Protocol*.

## Licence

To be confirmed. The proposal is **Apache-2.0** for code and **CC-BY-4.0** for the protocol specification.

---

<div align="center">

**Don't recycle. Cycle.** ♻️

*Your old screen isn't dead. It's just waiting for the next picture.*

</div>
