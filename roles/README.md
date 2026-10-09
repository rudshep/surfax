# Control roles
Roles give controls stable meanings independent of particular hardware.
Status: Not started.
Milestone: M5 — controls, descriptors, bindings, and event handling.
Roles are grouped into profiles such as `gamepad`, `media-remote`, `keyboard`,
and `pointer`. Use [`TEMPLATE`](TEMPLATE) to propose one role.
| Starter role | Purpose |
| --- | --- |
| `gamepad.south` | South face button |
| `gamepad.left_stick` | Left analog stick |
| `gamepad.right_stick` | Right analog stick |
| `gamepad.dpad` | Directional pad |
| `media.play_pause` | Toggle playback |
| `media.stop` | Stop playback |
| `media.next` | Next track or item |
| `media.prev` | Previous track or item |
| `media.volume` | Adjust volume |
| `kbd.key` | Keyboard key input |
| `pointer.primary` | Primary pointer action |
See [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md) for the project context.
