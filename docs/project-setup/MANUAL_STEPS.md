# Manual GitHub setup

The GitHub CLI is installed in the scaffold environment but is not authenticated.
An authorized repository maintainer must complete these steps.

## Repository settings

- Set the default branch to `main`.
- Set the description to: `Surface Facsimile: turn old screens and controllers into second displays and control surfaces. Protocol: RCSDP.`
- Enable Issues and Discussions; disable Wiki.
- Add topics: `esp32`, `android`, `vnc`, `display`, `second-screen`,
  `home-assistant`, `esphome`, `midi`, `hid`, `retro`, `repurpose`, `proxmox`.
- Set up the labels and milestones listed below.
- Use the copy-paste issue text in [ISSUES.md](ISSUES.md) to create the starter
  issues.
- Configure GitHub's private vulnerability reporting if it is not already
  enabled.

## Labels

Create these labels (names shown exactly):

- Type: `bug`, `enhancement`, `docs`, `question`, `adr`, `device-report`,
  `profile`
- Area: `protocol`, `agent`, `esp32`, `android`, `browser`, `proxmox`,
  `controls`, `bindings`, `hardware-bridge`, `home-assistant`
- Status: `good first issue`, `help wanted`, `needs-measurement`, `blocked`

## Milestones

Create these milestones with the descriptions shown:

| Milestone | Description |
| --- | --- |
| M0 | Measure stock ESP32 VNC viewer on a real CYD |
| M1 | Discovery (mDNS) |
| M2 | Pairing and persistent Surface number |
| M3 | Sessions (live desktop region on the CYD) |
| M4 | Touch |
| M5 | Controls (descriptor, virtual HID, JSON event stream, fail-safe release on disconnect) |
| M6 | Android 4 surface |
| M7 | Sources (Proxmox VM console, headless browser) |
| M8 | Tier 0 browser surface |

## Git initialization

The working branch is managed by the pull-request environment. Do not rename it
or rewrite its history here. When preparing a fresh repository independently,
initialize it on `main` and make the scaffold commit with message
`chore: scaffold repository structure`.
