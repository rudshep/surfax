# SURFAX on Android 4

> **Status:** plan, design phase. No code yet.
> **Part of the MVP.** Android 4 tablets are one of the headline "drawer devices", so they ship alongside the ESP32 CYD.

## 1. What "Android 4" means here

Android 4.x covers API levels 14 to 19 (4.0 to 4.4.4). They differ a lot, so we define tiers:

| Tier | Versions | Plan |
|---|---|---|
| **Supported** | 4.4 (API 19) | Native app. Browser fallback also works well. |
| **Supported** | 4.1 to 4.3 (API 16 to 18) | Native app. Browser fallback only via polled JPEG/MJPEG (no WebSocket, see below). |
| **Best effort** | 4.0.x (API 14 to 15) | Browser fallback only at first; native app if a contributor with a device steps up. |

*Why these lines?* Android's network service discovery API (the system mDNS client) arrived in API 16 ✅, and the WebView only switched to a Chromium engine at 4.4, which is also when it gained WebSocket support ✅.

## 2. Strategy: two doors in

1. **Zero-install door (Tier 0).** Open the host agent's URL in the tablet's browser and get a stream immediately. Built for MJPEG/polled JPEG so it works on old browsers.
2. **App door.** The same page offers "Install the SURFAX app". The agent serves the APK, the user allows installs from unknown sources, and the app takes over. The app gives proper discovery, full-screen kiosk behaviour, a stable ID and the control-surface descriptor.

The app is intentionally tiny: **one full-screen activity, a small RFB-subset viewer, and an RCSDP responder.** Nothing else.

### Why not just use a WebView?

* WebSocket doesn't exist in the WebView before 4.4 ✅.
* The 4.4 WebView is frozen at Chromium 30 and doesn't have full parity with Chrome, so modern pages may not render ✅. (Existing wall-panel apps document the same limitation.)
* A native app can enumerate hardware (touch, keys, sensors), hold the screen awake, and recover from Wi-Fi drops properly.

## 3. Obstacles we expect

Confidence tags: ✅ checked against public docs; 🧠 background knowledge, **verify on real devices**.

### 3.1 Getting the app onto the device

| Obstacle | Mitigation |
|---|---|
| The old Play Store is often unusable or outdated 🧠 | Sideload. The agent serves the APK; the user enables "unknown sources". Also document `adb install`. |
| Stock browsers may refuse or mishandle APK downloads 🧠 | Offer a QR code and a plain download link; document per-vendor quirks in profiles. |
| A user typing a URL on an old on-screen keyboard is painful | Agent prints a **short URL and a QR code**; the CLI can also show the LAN address. |

### 3.2 Building for it

| Obstacle | Mitigation |
|---|---|
| Modern Android build tools and libraries have raised minimum API levels 🧠 | **No third-party dependencies.** Plain Java against the platform SDK; set a low `minSdk` (target 16, stretch 14). |
| Old Java language level and a 64K-method limit before API 21 🧠 | Keep the app tiny so multidex is never needed. |
| Android Studio and emulator images for old APIs get harder to use 🧠 | Document a known-good build recipe; keep CI headless. |

### 3.3 Discovery

| Obstacle | Mitigation |
|---|---|
| `NsdManager` exists only from API 16 ✅ | For 4.0.x use a fallback beacon (below). |
| NsdManager is widely reported as flaky on some devices 🧠 | Ship a **UDP broadcast/multicast beacon fallback** that carries the same TXT-style fields. |
| Android filters multicast to save battery 🧠 | Hold a multicast lock while discovering. |
| Some routers block mDNS between Wi-Fi clients 🧠 | Document it; the beacon fallback and a manual "enter host address" screen cover the gap. |

### 3.4 Networking and security

| Obstacle | Mitigation |
|---|---|
| TLS 1.1/1.2 exist on API 16 to 19 but are **disabled by default** until API 20 ✅ | SURFAX does not depend on TLS. LAN plaintext plus pairing keys (see ARCHITECTURE §4.4). The app must never assume it can reach a modern HTTPS service. |
| Old root-certificate stores 🧠 | Another reason not to rely on public HTTPS for anything: updates and APKs come from the local host. |
| Wi-Fi is often 2.4 GHz only with weak antennas 🧠 | Test real throughput; downshift encodings aggressively. Offer Ethernet via USB adapter where supported. |
| Wi-Fi sleeps when the screen idles 🧠 | Hold a Wi-Fi lock and a screen-on window flag while attached. |

**Security reality check:** an Android 4 device has **many unpatched, well-known vulnerabilities** 🧠. Treat it as a hostile-grade appliance:

* Put it on a **separate VLAN or guest network** with no route to anything valuable.
* **Never sign in** to personal accounts on it, and don't browse the open web with it.
* SURFAX itself is for trusted LANs only. A dedicated SURFAX VLAN solves both problems.

### 3.5 Performance

| Obstacle | Mitigation |
|---|---|
| Slow single/dual-core ARMv7 CPUs, 512 MB to 1 GB RAM 🧠 | Keep draw paths simple: decode into reused bitmaps, blit dirty rectangles to a `SurfaceView`/canvas. Avoid per-frame allocation (old garbage collectors pause visibly). |
| Hardware H.264 decoders exist from API 16 but vary wildly by chipset 🧠 | Not on the MVP path. Use Raw/Hextile/JPEG tiles first, and measure. |
| Odd resolutions and aspect ratios (e.g. 1024×600, 800×480) 🧠 | The descriptor reports the true native size; the agent renders at exactly that. |

### 3.6 Kiosk behaviour

| Obstacle | Mitigation |
|---|---|
| No true lock-task kiosk mode on old Android (that arrived in API 21) 🧠 | Register as a **home/launcher** app so Home returns to SURFAX; start on boot; keep screen on; disable auto-lock where possible. |
| System/navigation bars can't always be hidden (immersive mode arrived at 4.4) 🧠 | Use the full-screen and low-profile flags available on older versions and accept a visible bar on some 4.0 to 4.3 tablets. |
| Vendors' custom launchers and task killers may stop the app 🧠 | Document per-device quirks in profiles. |
| Screen off/brightness control without root is limited 🧠 | Dim via window brightness; treat full screen-off as device-admin territory (research). |

### 3.7 Identity and hardware reality

| Obstacle | Mitigation |
|---|---|
| Hardware IDs are unreliable or restricted 🧠 | Generate a random ID at first run and store it. Pairing binds to that. |
| Batteries in old tablets can be degraded or **swollen**; running them plugged in 24/7 stresses them 🧠 | **Safety:** inspect for swelling before wall-mounting. Don't use a swollen device. Where the design allows, run on mains with the battery removed or replaced. |
| Screens may have burn-in or dead touch zones 🧠 | Profile a "usable area" and dim or rotate content. |

### 3.8 What an Android 4 surface exposes as controls

A tablet is mostly a screen, but it can still describe itself:

| Endpoint | Controls |
|---|---|
| Display | Native size, backlight (window brightness) |
| Pointer | Multi-touch if the panel supports it |
| Controller | Hardware keys (volume, sometimes back/home), accelerometer, light sensor, battery level. If a USB gamepad is attached via OTG, its controls appear via `DESCRIPTOR_CHANGED`. |

See [RCSDP-CONTROLS.md](RCSDP-CONTROLS.md).

## 4. MVP definition of done (Android 4)

1. On a real Android 4.1+ tablet, open the agent's URL, tap *Install*, and launch the app **without a computer cable or developer tools**.
2. The tablet shows "**Surface N**" and appears in the agent's list within seconds.
3. A session shows live content at the tablet's native resolution.
4. Touch works through the pixel plane.
5. It survives a Wi-Fi drop and a host restart without anyone touching the tablet.
6. It runs for 24 hours on mains power without being killed by the OS.

## 5. First experiments (before committing to the design)

| # | Experiment | Learn |
|---|---|---|
| A0 | Hello-world APK on 3 different real devices (one per API range) | Does the build recipe work? Does install-from-URL work? |
| A1 | NsdManager advertise + resolve, vs the UDP beacon | How reliable is each on real Wi-Fi? |
| A2 | Raw/Hextile viewer to a `SurfaceView` | Frame rate and GC behaviour |
| A3 | JPEG tile decode into a reused bitmap | Is JPEG the sweet spot here too? |
| A4 | Stock-browser Tier 0 page (MJPEG/polled) on 4.0 to 4.3 | Does zero-install work at all on the oldest devices? |
| A5 | 24-hour soak on mains power | Does it stay alive and connected? |

## 6. Help wanted: device test reports

Please open an issue with:

* Model, Android version and API level, screen size and resolution
* Does the stock browser open the Tier 0 page? Does it download an APK?
* Wi-Fi band and whether mDNS discovery works
* Anything that surprised you

Each report becomes a **device profile** that helps the next person with the same drawer find.

## 7. Sources

* Network service discovery API level: <https://developer.android.com/reference/kotlin/android/net/nsd/NsdManager>
* WebView/Chromium switch at 4.4 and WebSocket support: <https://crossbario.readthedocs.io/en/stable/Browser-Support.html>, <https://firt.dev/android-4.4>
* WebView frozen at Chromium 30, limitations for dashboards: <https://github.com/thanksmister/wallpanel-android>
* TLS 1.1/1.2 disabled by default on API 16 to 19: <https://github.com/square/okhttp/issues/2372>, <https://github.com/xamarin/xamarin-android/issues/1615>
