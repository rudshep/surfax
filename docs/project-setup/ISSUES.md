# Starter issues

Copy these into GitHub Issues after applying the labels and milestones in
[MANUAL_STEPS.md](MANUAL_STEPS.md). Labels and milestones are suggestions for
the corresponding issue.

## M0: Measure a stock ESP32 VNC viewer on a real CYD

**Milestone:** M0  
**Labels:** `needs-measurement`, `esp32`

Measure frame rate and free heap for Raw and Hextile with a stock ESP32 VNC
viewer on a real Cheap Yellow Display. Write up the device, test conditions,
measurements, and limitations. Decide whether an RFB subset is sufficient or a
private JPEG-tile encoding is needed.

## Decide licence

**Milestone:** M0  
**Labels:** `adr`, `question`

Discuss licensing. Proposal: Apache-2.0 for code and CC-BY-4.0 for the protocol
specification. Do not add a LICENSE file until a decision is made.

## Decide RCSDP wire format

**Milestone:** M1  
**Labels:** `adr`, `protocol`

Compare JSON, CBOR, and compact binary for discovery, descriptors, and events.
Record trade-offs and evidence before choosing.

## Android experiments A0-A5

**Milestone:** M6  
**Labels:** `android`, `needs-measurement`

- [ ] Build and run a hello-world APK on 3 real devices.
- [ ] Compare `NsdManager` with a UDP beacon.
- [ ] Test a Raw/Hextile viewer rendering to `SurfaceView`.
- [ ] Test JPEG tile decode into a reused bitmap.
- [ ] Test a stock-browser MJPEG/polled page on Android 4.0 to 4.3.
- [ ] Run a 24-hour soak on mains power.

Record exact models, OS/API versions, test setup, and measurements.

## Seed the roles registry

**Milestone:** M5  
**Labels:** `good first issue`, `controls`

Review the starter roles in `roles/README.md` and propose role definitions and
examples using `roles/TEMPLATE`.

## Add your first device profile: open your drawer

**Milestone:** M2  
**Labels:** `good first issue`, `profile`, `device-report`

Test a device from your drawer and contribute a profile based on
`profiles/TEMPLATE`. Include the exact model, OS or firmware version, and test
date.

## Name check: Surfaxe (an unrelated Python materials-science package) and package-name availability for SURFAX/RCSDP

**Milestone:** M0  
**Labels:** `question`, `docs`

Check possible name conflicts with Surfaxe (an unrelated Python
materials-science package) and check package-name availability for SURFAX and
RCSDP. Record sources and findings.

## Windows and macOS virtual HID options for control bindings

**Milestone:** M5  
**Labels:** `bindings`, `needs-measurement`

Research practical virtual HID options on Windows and macOS. Compare platform
support, permissions, maintenance, and licensing; record sources and evidence.
