# Contributing to SURFAX

SURFAX is in the design phase. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
for the project direction.

The easiest first contribution: **open your drawer**. File a device test report
or contribute a profile based on `profiles/TEMPLATE`.

Other routes: ESP32, Android, networking, Linux, web, Home Assistant/ESPHome,
MIDI/HID/OSC, and documentation.

Ground rules:
- Keep pull requests small and focused on one idea.
- Measurements beat opinions.
- Name the exact model and OS version behind every device claim.
- Be kind and constructive.

Licensing hygiene: study, but do not copy code from AGPL/GPL projects (for
example, RustDesk or Sunshine). Re-implement from the protocol description and
say so in your pull request.

Safety: SURFAX is for trusted networks. Never route emergency stops or
timing-critical signals through it.
