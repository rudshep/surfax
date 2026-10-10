# RCSDP: Screens *and* Controls

> **Status:** draft v0, design phase. Names and fields are proposals.
> **Related:** [ARCHITECTURE.md](ARCHITECTURE.md) · [PRIOR_ART.md](PRIOR_ART.md)

## 1. The idea

A modern monitor says "I'm a screen, this big." A modern USB controller says "I have these buttons and sticks." A SURFAX surface should be able to say **both**, in one introduction:

> *"I'm Surface 7. I have a 320×240 screen, a touch panel, two encoders with push buttons, four buttons, a thumb joystick and eight LEDs."*

The host then treats it the way a person would: **screen, keyboard, mouse and controller.**

* The screen becomes a display session (pixels over the RFB plane).
* Touch becomes a pointer.
* Keys become a keyboard.
* Knobs, sliders, joysticks and buttons become a **controller** that software on the host can use, either as a standard virtual device or through a simple event API.
* LEDs, haptics and per-control labels flow the other way.

Three shapes we are designing for:

| Device | Screen | Controls |
|---|---|---|
| **Synth module** | Scope, patch map, parameter page | Encoders (with push), buttons, LEDs |
| **Robotics controller** | Camera view, telemetry | Two joysticks, d-pad, triggers, buttons |
| **Audio player** | Album art, track info | Play/pause/stop/next/prev, volume knob or slider |

## 2. The model

```
Surface
 ├── Display endpoints   (0..n screens)
 ├── Pointer endpoint    (touch / mouse-like)
 ├── Keyboard endpoint   (keys)
 └── Controller endpoint (everything else: knobs, sliders, sticks, buttons, LEDs, sensors)
        └── Controls (each one: id, kind, range, role, label, ...)
```

* An **endpoint** is a group of related capability.
* A **control** is one physical or logical thing a person can touch, turn, push or see (a knob, an LED).
* A control may be **input** (surface → host), **output** (host → surface), or both (a motorised fader).

## 3. Control kinds

| Kind | Value | Typical hardware | Reliability class (see §6) |
|---|---|---|---|
| `button` | pressed / released | Tactile switch, arcade button | **edge** |
| `toggle` | on / off (latching) | Switch | **edge** |
| `encoder` | relative steps, optional push | Rotary encoder | **delta** |
| `knob` | absolute 0..N | Potentiometer | **level** |
| `slider` | absolute 0..N | Fader | **level** |
| `joystick` | two axes, optional click | Thumb stick | **level** (+ edge for click) |
| `hat` | 8-way direction | D-pad | **edge** |
| `trigger` | absolute 0..N | Analog shoulder trigger | **level** |
| `key` | pressed / released + code | Keyboard key | **edge** |
| `matrix` | grid of buttons | Keypad, pad grid | **edge** |
| `strip` | absolute position | Touch strip, wheel | **level** |
| `pointer` | x, y, pressure, contact ID | Touchscreen | **edge** for contact, **level** for position |
| `sensor` | numeric reading | Accelerometer, light, battery | **level** |
| `led` / `rgb` | on-off, brightness, colour | LED, LED ring | output |
| `haptic` | pulse / pattern | Vibration motor | output |
| `backlight` | 0..100% | Screen backlight | output |
| `label` | short text | Per-control OLED, scribble strip | output |

## 4. Properties of a control

| Property | Meaning |
|---|---|
| `id` | Stable path-like ID within the surface, e.g. `enc/1`, `joy/left` |
| `kind` | One of the kinds above |
| `label` | Human-friendly name |
| `range` | Min, max, optionally step and **resolution in bits** (so a 7-bit pot isn't pretended to be 16-bit) |
| `role` | Semantic role, e.g. `gamepad.left_stick`, `media.play_pause`, or none (see §5) |
| `group` | Visual or logical grouping, e.g. `left-hand`, `transport` |
| `pos` | Optional x/y on the device, so the host can draw an on-screen picture of the surface |
| `access` | `in`, `out` or `inout` |
| `attach` | Optional link to a display zone, for controls with their own little screen |
| `rate` | Suggested maximum update rate for continuous controls |

## 5. Roles and profiles

HID taught us that self-description alone gives numbers without meaning: a "button 3" could be anything. So SURFAX adds a **semantic layer** on top.

* A **role** says what a control *is for*. Roles are dotted names from a small community-maintained registry (`roles/` in the repo): `gamepad.south`, `gamepad.left_stick`, `gamepad.dpad`, `media.play_pause`, `media.stop`, `media.volume`, `kbd.key`, `pointer.primary`, and so on.
* A **profile** is a named bundle of roles, borrowed from the MIDI-CI idea. If a surface declares it satisfies the `gamepad` profile, the host knows it can safely present it as a standard gamepad. If it satisfies `media-remote`, the host can bind it to media keys.
* Controls with **no role** are still fully usable: they appear as generic controls in the host API.
* Roles are **data files**, so adding one is an easy pull request.

## 6. Reliability classes

Borrowed from how remote-desktop and game-streaming stacks handle touch and controllers: losing a "down" or "up" leaves things stuck, but losing a joystick sample doesn't matter.

| Class | Examples | Rule |
|---|---|---|
| **Edge** | button down/up, hat change, touch contact start/end | Never drop, never reorder |
| **Delta** | encoder steps | May be summed together, never dropped |
| **Level** | knob, slider, stick position, sensor | Latest value wins; may be rate-limited |

**Fail-safe rule:** if the link drops, the host must synthesise **releases for every pressed edge control and re-centre every level control it has bound as a stick**. No stuck buttons or runaway joysticks, ever.

## 7. Discovery flow

1. **Advertise (small).** The surface announces `_rcsdp._tcp` via mDNS with a few TXT keys: version, ID, name, model, plus summary counts (`disp`, `ctl`) and a **descriptor hash** `cd`.
2. **Connect and HELLO.** The host connects to the control port and gets the basic descriptor.
3. **Fetch (detailed, on request).** The host asks for the full **control descriptor** if it doesn't already have one cached for that hash. (Same pattern as OSCQuery: advertise tiny, describe in detail.)
4. **Bind.** The host agent creates bindings (see §8) for the roles it understands.
5. **Hot-plug.** If the control set changes (say a USB gamepad is attached to an Android tablet), the surface sends `DESCRIPTOR_CHANGED` with a new hash.

If a surface can't self-describe (a stock tablet that only has touch and volume keys), the host derives the descriptor from its **device profile** by model ID, and the surface adds anything it can auto-detect.

## 8. Host projections (bindings)

The point of the abstraction: software on the host shouldn't care that the knob is on a CYD or a tablet.

| Projection | What the host sees | Phase |
|---|---|---|
| **Virtual HID (Linux uinput)** | A normal gamepad/keyboard/mouse; works with games and desktop apps | **MVP** |
| **Event stream (JSON over a local socket/WebSocket)** | Simple "control X changed to Y" events plus commands for outputs. The universal fallback and the easiest API. | **MVP** |
| **OSC / OSCQuery** | Controls appear as an addressable OSC tree, discoverable the way many creative tools expect | After MVP |
| **MIDI (virtual port)** | Knobs become CCs, buttons become notes; later MIDI 2.0 UMP | After MVP |
| **ROS 2 `Joy`** | Sticks and buttons as a joystick message | After MVP |
| **Home Assistant / MQTT** | Controls as entities | After MVP |
| **Virtual HID (Windows / macOS)** | Same as Linux, harder | Needs research |

Outputs go the other way: an app sets an LED, a label or a vibration through the same API.

**Pointer shortcut:** touch on a *display* endpoint goes over the pixel plane as ordinary RFB pointer events, which is the simplest thing and works with any VNC server. Richer controls use the RCSDP input channel. We deliberately avoid sending the same touch twice.

## 9. Worked examples

### 9.1 Cheap Yellow Display (the MVP reference surface)

| Endpoint | Controls |
|---|---|
| Display | 320×240, RGB565, backlight (output) |
| Pointer | Resistive touch, single contact |
| Controller | Typically also a BOOT button, an RGB LED and a light sensor *(check per board variant)* |

### 9.2 Synth module

| Endpoint | Controls |
|---|---|
| Display | Small colour screen showing a scope or parameter page |
| Controller | 4× `encoder` (with push), 6× `button`, 8× `led`, 1× `rgb` |

Host binds: encoder turns → OSC or MIDI CC; buttons → notes or transport; LEDs driven by the patch state. The screen is rendered by software on the host. Audio-rate or timing-critical signals **never** travel over SURFAX; it is the display-and-control layer.

### 9.3 Robotics controller

| Endpoint | Controls |
|---|---|
| Display | Camera view and telemetry |
| Controller | 2× `joystick` (with click), 1× `hat`, 4× face `button`, 2× `trigger`, `sensor` battery, `haptic` |

Host binds: `gamepad` profile → virtual gamepad or ROS `Joy`. The fail-safe rule in §6 matters here. Emergency stops must be **independent hardware**, never routed through SURFAX.

### 9.4 Audio player

| Endpoint | Controls |
|---|---|
| Display | Album art and track info (host-rendered) |
| Controller | `button` ×5 (`media.play_pause`, `media.stop`, `media.next`, `media.prev`, and a mode key), `knob` or `encoder` (`media.volume`), `led` for state |

Host binds: `media-remote` profile → the OS media keys, so *any* player responds. The display source renders now-playing and album art.

## 10. Illustrative descriptor (not final)

The wire format is decided in milestone M1; JSON is the leading candidate because it is easy to read and debug. This is purely to make the shape concrete.

```json
{
  "rcsdp": 0,
  "id": "a1b2c3",
  "name": "Bench synth module",
  "displays": [
    { "id": "main", "w": 320, "h": 240, "fmt": ["rgb565"], "enc": ["raw", "hextile"] }
  ],
  "controls": [
    { "id": "enc/1", "kind": "encoder", "push": true, "label": "Cutoff", "role": "synth.param" },
    { "id": "btn/1", "kind": "button", "label": "Play", "role": "media.play_pause" },
    { "id": "led/1", "kind": "led", "access": "out" }
  ],
  "profiles": []
}
```

## 11. Limits and honesty

* **Latency.** Wi-Fi jitter makes this unsuitable for anything timing-critical. Wired transports (Ethernet, USB) are on the long-term wish list.
* **Safety.** Never use it as the safety path for motion or power. Emergency stops stay hardware.
* **Not a USB replacement.** If you can plug in a real USB controller, you should.
* **Scope.** MVP covers touch, buttons, encoders, sticks and LEDs on a handful of surfaces, bound to Linux virtual input and a JSON event stream. The rest is roadmap.

## 12. Open questions

1. Final wire format for the descriptor and for events (JSON, CBOR, or a compact binary framing).
2. Should the input channel be TCP only, or add a UDP path for low-jitter analog streams?
3. How many roles should ship in v0? (Suggest: gamepad, media, keyboard, pointer.)
4. How do multi-display surfaces (a controller with two screens) map to host sessions?
5. What is the right model for **absolute versus relative** knobs when the host and the physical position disagree (motorised faders, soft takeover)?
6. What is the state of virtual gamepad drivers on Windows and user-space virtual HID on macOS today? Needs research.
