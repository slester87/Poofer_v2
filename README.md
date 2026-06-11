# Poofer_v2

![Poofer live fire](Assets/Images/blurred_poof.jpg)
![Poofer mushroom poof](Assets/Images/mushroom_poof.jpeg)

![CI](https://github.com/slester87/Poofer_v2/actions/workflows/ci.yml/badge.svg?branch=development)

Firmware and UI for a two-poofer networked control system built from one ESP32-C3 + WiSeFire node per poofer.

The normative v2 implementation target is documented in [DESIGN.md](DESIGN.md).

## V2 Architecture Baseline

- Exactly two poofer nodes are supported in v2.
- Nodes persist one assigned role locally: `stage-left`, `stage-right`, or `unassigned`.
- The browser is the single active controller for the show.
- A dedicated external Wi-Fi network is required. The browser and both nodes join that network as clients.
- Nodes are discovered over `mDNS`, with manual fallback allowed later if needed.
- The browser talks directly to each node over websocket connections.
- The operator UI exposes three fire targets:
  - `stage-left`
  - `both`
  - `stage-right`
- `both` is one logical command target with best-effort fan-out to both leaf nodes.
- Safety is hybrid:
  - local node faults inhibit only that node
  - a global browser-side `Armed` gate blocks all outgoing fire commands when false
- Firing remains hold-based:
  - `DOWN`
  - `HOLD`
  - `UP`
- Missing a poof is acceptable. A stale or unintended poof is not.

## Setup Mode

`Poofer ID Setup` is a dedicated UX for assigning and reassigning `stage-left` and `stage-right`.

These rules are mandatory in setup mode:

- Entering setup mode automatically forces `Armed = false`.
- Firing controls are disabled while setup mode is active.
- Role conflict detection automatically redirects into setup mode.
- Entering setup mode does not preserve a prior armed state.
- Exiting setup mode does not re-arm the system automatically.

Setup mode is triggered when:

- a node is `unassigned`
- both discovered nodes claim the same role
- one of the required roles is missing
- the operator explicitly chooses `Poofer ID Setup`

## Quick Start

For a full setup guide, see `INSTALL.md`.

Condensed steps:

- Install ESP-IDF 5.5.x and ensure `idf.py` is on your PATH.
- Optional: copy `.env.example` to `.env` to set defaults like `POOFER_SERIAL_PORT`.
- Have an existing dedicated Wi-Fi network ready before node setup. `Poofer_v2` does not host its own control AP.
- Build firmware for each poofer node: `python3 scripts/build.py`
- Flash each node: `python3 scripts/flash.py --port /dev/cu.usbmodemXXXX`
- Provision both nodes onto the same dedicated external Wi-Fi network
- Open the browser control console and complete `Poofer ID Setup`
- Verify one node is `stage-left` and one is `stage-right`
- Intentionally arm the system before live operation

## Hardware

- ESP32-C3 Super Mini dev board: https://www.amazon.com/dp/B0D4QK5V74
- Solenoid: https://www.amazon.com/dp/B00DQ1J4H0
- Power supply (12V): https://www.amazon.com/dp/B0D9D5L3B5
- One WiSeFire-driven poofer node per effect head
- Dedicated external Wi-Fi AP/router for the poofer network
- Full Bill of Materials: `BOM.md`

## System Model

`Poofer_v2` is a two-node system:

- one browser control console
- one `stage-left` poofer node
- one `stage-right` poofer node
- one dedicated external Wi-Fi network that all three join as clients

Each node persists one role locally:

- `stage-left`
- `stage-right`
- `unassigned`

The browser discovers nodes over `mDNS`, opens direct websocket sessions to them, and exposes three live fire targets:

- `stage-left`
- `both`
- `stage-right`

`both` is one logical operator target with best-effort fan-out to the two nodes.

## Safety Model

The safety model is hybrid:

- each node owns its own local firing and inhibit state
- the browser owns a global `Armed` gate for outgoing fire commands
- better to miss a poof than to poof when not intended

Fire control is hold-based:

- `DOWN` starts a fire interaction
- repeated `HOLD` messages keep it alive
- `UP` ends it

If command liveness is lost, the node must stop locally.

## Setup And Deployment

Normal live use assumes exactly one valid left/right pairing. If deployment is ambiguous, the system must move into `Poofer ID Setup`.

Setup mode is used for:

- initial role assignment
- reassignment
- conflict resolution

Setup mode is triggered when:

- a node is `unassigned`
- both nodes claim the same role
- one required role is missing
- the operator explicitly requests setup

The browser UI enforces this by forcing setup mode and clearing `Armed` until exactly one connected
`stage-left` node and exactly one connected `stage-right` node are present.

In setup mode:

- `Armed` is forced false
- firing controls are disabled
- exiting setup does not automatically re-arm the system

## Operator UI

The live console is intentionally simple:

- one `1x1` button for `stage-left`
- one `2x1` center button for `both`
- one `1x1` button for `stage-right`
- one global `Armed` slider
- one `Poofer ID Setup` entry point

The control buttons retain the v1-style three-second depletion gauge.

Per-node `last-held` is shown for:

- `stage-left`
- `stage-right`

`both` does not fabricate an aggregate hold duration.

## Status Colors

Status colors are shared between the browser UI and each node's first WS2812 pixel:

- `green` = ready / good-to-go
- `blue` = disconnected
- `red` = connected but unavailable, faulted, or inhibited
- `orange` = actively firing

When `Armed` is false, buttons should be muted and blurred rather than using a different status color. Color communicates node state; the armed slider communicates whether firing is allowed.

## Networking

The network model in v2 is explicit:

- bring your own dedicated Wi-Fi network
- do not use a poofer node as the AP host
- do not use LoRa in v2
- discover nodes with `mDNS`
- control nodes with direct browser-to-node websocket sessions

## Web UI

The browser UI is a multi-node control console. It keeps a browser-side registry of known node origins,
opens one WebSocket per node, maps node state by reported role, and fans out `both` as role-local
commands to the eligible `stage-left` and `stage-right` nodes.

Automatic mDNS enumeration is not available directly from ordinary browser JavaScript. The setup UI
therefore provides a node-origin editor, stores additional origins in browser `localStorage` under
`poofer_v2_node_origins`, and can assign each connected node's persisted role through `/api/role`.

## Protocol Summary

The node WebSocket protocol is JSON-only in v2. Legacy singleton text commands such as `DOWN`, `UP`, and `PING` are not accepted by this firmware.

Implemented message families:

- `claim` establishes the browser controller/session for one node.
- `fire` sends `DOWN`, repeated `HOLD`, and `UP` phases with a `command_id`.
- `ping` requests a status refresh.
- `ack` reports per-command acceptance or rejection.
- `state` publishes node identity, role, ownership, readiness, firing state, and hold timing.

Fire-command shape:

```json
{
  "type": "fire",
  "phase": "DOWN",
  "command_id": "cmd-uuid",
  "controller_id": "browser-uuid",
  "session_id": "session-uuid",
  "target": "stage-left",
  "expires_in_ms": 500,
  "sent_at_ms": 1234567890
}
```

The current browser UI uses this protocol against every configured node origin and performs best-effort fan-out from the browser.
Nodes accept fire commands only for their own assigned role (`stage-left` or `stage-right`). The `both` target is a browser-level fan-out concept; it is not sent to a node as a physical output target.

The normative protocol requirements live in [DESIGN.md](DESIGN.md).

## Configuration

The v2 configuration model is:

- each node persists its role in non-volatile storage
- each node has a stable hardware identity
- browser setup flow assigns or reassigns node roles
- browser remains the single controller in live operation

For implementation details and acceptance criteria, use [DESIGN.md](DESIGN.md).

## Development

- Linting entry point: `scripts/lint.sh`
- Git hooks: `pre-commit install`

## Releases

Firmware artifacts are built in CI for tags matching `fw-*`.

The workflow uploads these files as artifacts:

- `bootloader.bin`
- `partition-table.bin`
- `poofer.bin`
- `spiffs.bin`
- `flasher_args.json` and `*_flash_args` helpers

Create a release tag:

```bash
scripts/release_tag.sh 1.0.0
```

## Config

Optional local config is supported via `.env`.

- Copy `.env.example` to `.env`
- Set `POOFER_IDF_PATH`, `POOFER_SERIAL_PORT`, and any optional overrides

## Safety

- Better to miss a poof than to poof when not intended.
- Setup and configuration are always non-firing modes.
- Browser reload or disconnect must resolve to disarmed or locally stopped behavior.
- Nodes must reject stale, duplicate, expired, or ownership-ambiguous fire commands.

## License

MIT. See `LICENSE`.
