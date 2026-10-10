# SURFAX Prior Art

> **Status:** research notes, October 2026. Not a complete survey.
> **Goal:** learn from proven protocols *without* becoming a remote-desktop product.

SURFAX is not remote desktop. We care about two narrow things that every remote-desktop and game-streaming protocol had to solve:

1. **Frame buffer handling:** how pixels get from a host to a (possibly weak) client.
2. **Input and device handling:** how touch, keys, mice and controllers get back, and how devices describe themselves.

Everything below is read through that lens.

**How to read the confidence tags.** Items marked ✅ were checked against public docs while writing this. Items marked 🧠 come from background knowledge and were *not* re-checked. Please verify those before relying on them, and open a PR if we got something wrong.

---

## 1. At a glance

| | **VNC / RFB** | **Microsoft RDP** | **Sunshine / Moonlight** | **RustDesk** |
|---|---|---|---|---|
| **Main purpose** | Remote framebuffer | Remote Windows sessions | Low-latency game streaming | Remote support / TeamViewer alternative |
| **Pixels** | Rectangles, client picks encodings and pixel format | Channel-based; bitmap/RemoteFX/graphics-pipeline extensions ✅ | Hardware video codec (H.264/HEVC) over RTP ✅ | VP8/VP9/AV1 and H.264/H.265 ✅ |
| **Input** | Pointer and key events 🧠 | Core input, plus an input channel with touch (MS-RDPEI) ✅ | Mouse, keyboard, controller, touch, pen over a control stream ✅ | Mouse and key events as protobuf messages ✅ |
| **Extensibility** | Pseudo-encodings 🧠 | Static and dynamic **virtual channels** ✅ | Extra ENet channels, protocol extensions ✅ | Services plus a plugin system ✅ |
| **Pairing / identity** | Password per server 🧠 | Windows accounts 🧠 | **PIN pairing**, then HTTPS for paired clients ✅ | ID derived from a key pair, rendezvous server ✅ |
| **Dumb-client friendly?** | **Yes** (can be tiny) | No | No (needs a hardware decoder) | No |
| **Licence note** | Many implementations | Specs are public ✅ | Believed GPL-3.0 🧠 | AGPL-3.0 ✅ |

**Short version:** RFB is the only one light enough for an ESP32. The others are where we steal *ideas*: channels, pairing, input handling, feedback.

---

## 2. Frame buffer handling

### 2.1 VNC / RFB (revisited)

Already covered in [ARCHITECTURE.md §3.4](ARCHITECTURE.md). The properties that matter most for us: the client picks pixel format and encodings, updates are pull-based (free backpressure), and rectangles plus CopyRect make damage-based updates cheap.

### 2.2 Microsoft RDP

What the public specifications tell us:

* RDP is built on **channels**. The main connection carries core input and graphics, then a set of negotiated *static virtual channels* sits alongside it, and one of those multiplexes many *dynamic virtual channels*. ✅
* Features are separate extension specs, each a channel: graphics pipeline, RemoteFX codec, display update (resize), input (touch), clipboard, audio, file system, and more. ✅
* RDP provisions for a very large number of channels, so it was designed to be extended. ✅
* There is a lean "fast-path" encoding for input and output alongside the older "slow-path", and newer servers require clients to advertise fast-path support. ✅

**What we take:**

* **Channels per concern.** Display, input and device-control travel as separate logical channels, so a weak surface can ignore what it doesn't use.
* **Capability exchange up front.** Client and server agree what's supported before data flows.
* **The display-update channel is a precedent** for "the client tells the server its size and the server adapts", which is exactly SURFAX's descriptor-to-resolution idea.

**What we avoid:** the codec and graphics-pipeline stack. It is far too heavy for a CYD, and it assumes a Windows-style GDI world.

### 2.3 Sunshine / Moonlight

Moonlight is the client; Sunshine is an open-source host that speaks the same protocol. What the docs show:

* **Six protocols in one system:** an unencrypted HTTP port for exchanging public information to start pairing, an HTTPS port only available to paired clients, RTSP to negotiate ports and settings, an encrypted ENet control stream, RTP video, and RTP audio. ✅
* The control stream carries **input and extra stream information** over reliable UDP (ENet). ✅
* Video is H.264 or HEVC; audio is Opus. ✅
* Hosts run an **adaptive bitrate** system. ✅
* Clients exist on very odd hardware: someone ported a client to PlayStation 3 homebrew, using the PIN-challenge pairing handshake. ✅

**What we take:**

* **Pair with a short PIN or code**, then trust the pair. Same UX instinct as our on-screen number.
* **Separate "who are you" (open, public) from "do stuff" (authenticated)** endpoints.
* **Documented protocols attract ports** to strange hardware. That's a good argument for writing the RCSDP spec early and clearly.

**What we avoid:** hardware video codecs as a hard requirement. A thirteen-year-old tablet *may* have a hardware H.264 decoder, but an ESP32 certainly doesn't.

### 2.4 RustDesk

* A Rust core handles networking, encryption and codecs; the UI is separate. ✅
* **Services** (video, audio, input, clipboard) are independent and subscribed per connection. ✅
* A **rendezvous server** keeps a registry of online peers, each identified by an ID derived from its key pair; peers then try a direct connection via hole punching, with relay as fallback. ✅
* A QoS engine adjusts frame rate and bitrate from measured round-trip latency. ✅
* On Linux it injects input at a low level through `/dev/uinput`, because Wayland restricts higher-level injection. ✅
* It is licensed **AGPL-3.0**. ✅

**What we take:**

* **Stable identity from a key pair** is a good model for a surface ID.
* **RTT-driven quality adaptation** maps directly to our "downshift when Wi-Fi is bad" requirement.
* **uinput** confirms the Linux path for host-side virtual input devices.

**What we avoid:** internet rendezvous and NAT traversal (SURFAX is LAN-only), and *copying code*: AGPL has strong obligations that would affect SURFAX's own licence.

### 2.5 Others (background only 🧠)

| Project | Relevance |
|---|---|
| **SPICE** (used by Proxmox/QEMU) | Many channels (main, display, inputs, cursor, audio, USB redirect). Same "channel per concern" lesson, and USB redirection is a precedent for device passthrough. |
| **Looking Glass** | Shares frames from a VM to the host through shared memory. Interesting for the Proxmox-adjacent path, but not a network protocol. |
| **Parsec** | Proprietary low-latency streaming. Useful as a UX benchmark only. |
| **USB/IP** | Exports raw USB devices over a network. A "raw passthrough" alternative to our abstracted controls. |

---

## 3. Input and HID handling

### 3.1 What the streaming protocols send

Moonlight's client library exposes input as distinct message types over the control stream: ✅

| Input | Carried data |
|---|---|
| Mouse (relative) | delta X/Y |
| Mouse (absolute) | x, y **plus a reference width and height** |
| Keyboard | key code, action, modifiers |
| Controller | button flags, triggers, sticks |
| Touch | event type, pointer ID, x, y, pressure |
| Pen | tool type, buttons, tilt |

Observations:

* **Absolute pointer + reference size** is the right way to map a touch on a small screen to any host resolution. We should copy this.
* Sunshine has grown **feedback and rich-controller support** over time: emulated DualShock 4 with touchpad, motion sensors, battery state and LED control, plus rumble feedback going back to the client, and up to 16 gamepads. ✅
* Sunshine added **input batching** to cut latency. ✅

### 3.2 Edge events vs continuous events

A recent RDP server implementation (lamco-rdp-server) documents a design choice that applies directly to us: touch contact frames are **never coalesced or dropped**, because losing a "down" or "up" transition would leave a contact stuck. ✅

This gives a general rule we adopt in [RCSDP-CONTROLS.md](RCSDP-CONTROLS.md):

* **Edges** (press, release) must never be lost.
* **Continuous values** (a knob position) can be coalesced to the latest value.
* **Relative deltas** (an encoder turn) can be summed but not dropped.

### 3.3 Host-side virtual devices

To make a remote control act like a local one, the host needs a virtual input device. Linux has **uinput** ✅. Windows and macOS need drivers or user-space HID APIs and are harder (🧠 and to be researched; we have not verified the current state of virtual-gamepad drivers on Windows).

---

## 4. Describing devices and their controls

This is where RCSDP's *control surface* model needs prior art, because streaming protocols mostly assume **a client that is a known controller type** (a standard gamepad, a mouse). We want an arbitrary box with arbitrary knobs.

| Standard | How it describes controls | Lessons for us |
|---|---|---|
| **USB HID report descriptors** ✅ | The device describes the format and meaning of its reports using *usages* grouped into *usage pages*. | Self-describing hardware is a solved idea. But individual buttons "generally aren't assigned any meaning in HID", and parsers need **quirk tables** (one gamepad even lists its buttons backwards). Self-description is necessary but not sufficient, so we add semantic **roles** and community **profiles**. |
| **MIDI 2.0: MIDI-CI** ✅ | Bidirectional capability inquiry: *Profile Configuration* (preset behaviours), *Property Exchange* (detailed info like parameter names), and *Protocol Negotiation*, with fallback to MIDI 1.0. | The closest conceptual match to what we want. Steal the **profile** idea (a named bundle of expected controls), the **two-way confirmation**, and **graceful fallback**. |
| **OSCQuery** ✅ | A device advertises over mDNS, then serves an HTTP/JSON **tree** of nodes, each with a type, range, access (read/write) and description. | The closest *implementation* match. The pattern "advertise tiny, describe in detail on request" is exactly what we need, because mDNS TXT records are too small for a full control list. |
| **Linux evdev / SDL / browser Gamepad API** 🧠 | Host-side abstractions that apps already consume. | Tells us what the *host projections* should look like: if SURFAX controls appear as standard devices, existing software just works. |
| **ROS `Joy` messages** 🧠 | An array of axes and buttons. | A natural projection for robotics. |

---

## 5. Lessons we adopt

| # | Lesson | Source |
|---|---|---|
| 1 | Separate logical **channels** for display, input and device control | RDP, SPICE, RustDesk services |
| 2 | **Handshake, then capabilities**, before sending data | RDP, MIDI-CI, RFB encodings |
| 3 | **Short-code pairing**, then trust | Moonlight |
| 4 | Absolute pointer plus **reference size** | Moonlight |
| 5 | **Edge / delta / level** reliability classes | RDP touch handling, Sunshine batching |
| 6 | A **feedback path** host→device (LEDs, haptics, labels) | Sunshine rumble/LED |
| 7 | Self-description plus **roles and quirk profiles** | HID, MIDI-CI Profiles |
| 8 | **Advertise small, describe on request**, cache by hash | OSCQuery |
| 9 | **RTT-driven quality adaptation** | RustDesk, Sunshine ABR |
| 10 | **Never put a hardware codec on the critical path** | Reality of ESP32 and old tablets |

## 6. Where SURFAX is different

* None of the above treat the client as a **dumb, numbered, self-describing surface** that you plug in like a monitor.
* None unify **display endpoints and control endpoints** in one descriptor, with automatic binding to host-side virtual devices.
* None target **ESP32-class and Android 4-class** clients.

*Caveat:* this is based on a focused look, not an exhaustive code search. If you know a project that already does some of this, open an issue so we can collaborate rather than duplicate.

## 7. Licensing hygiene

* **Read specs freely.** RDP's specifications are public; MIDI, HID and OSCQuery are public references.
* **Study, don't copy, AGPL/GPL code.** RustDesk is AGPL-3.0 ✅. Sunshine and its client library are believed to be GPL-3.0 🧠. Copying code from either would pull SURFAX into the same licence terms. Re-implement from the protocol description instead, and record in PRs that you did so.
* **Verify before relying.** Please check licences yourself before borrowing any code.

## 8. References

* Moonlight protocol overview: <https://games-on-whales.github.io/wolf/stable/protocols/index.html>
* moonlight-common-c overview: <https://deepwiki.com/moonlight-stream/moonlight-common-c>
* Sunshine release notes (input features): <https://github.com/LizardByte/Sunshine/releases/tag/v0.21.0>
* Moonlight PS3 client (example of a port to odd hardware): <https://github.com/Cruslan/PS3-Moonlight/releases/tag/v1.0.0>
* RustDesk architecture overviews: <https://pyshine.com/RustDesk-Open-Source-Remote-Desktop/> and <https://openapps.pro/apps/rustdesk>
* Microsoft RDP overview: <https://learn.microsoft.com/windows/win32/termserv/remote-desktop-protocol>
* RDP channel hierarchy: <https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpemt/4d98f550-6b0d-4d5f-89f5-2ac8616246a2>
* RDP specification index: <https://rdsgurus.com/rdp-protocol-specifications-and-other-helpful-documentation/>
* lamco-rdp-server v1.4.5 (touch handling): <https://github.com/lamco-admin/lamco-rdp-server/releases/tag/v1.4.5>
* HID descriptors explained: <https://web.dev/hid>
* HID in Unity's Input System: <https://docs.unity3d.com/Packages/com.unity.inputsystem@1.1/manual/HID.html>
* ESP-IDF HID descriptor parser and quirks: <https://components.espressif.com/components/badgeteam/hid-host/versions/0.3.1/readme>
* MIDI 2.0 core specs (MIDI-CI, Profiles, Property Exchange): <https://midi.org/?p=1384>
* OSCQuery implementations: <https://npmjs.com/package/oscquery>, <https://www.nuget.org/packages/OscQueryLibrary/2.0.0>
