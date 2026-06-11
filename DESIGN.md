# Poofer_v2 Design Spec

This document is the normative design spec for `Poofer_v2`.

The goal is to describe the v2 system precisely enough that a new implementation can be built from this document alone without relying on undocumented assumptions from `Poofer`.

If code and this document disagree during early development, prefer this document unless an explicit design update changes it.

## 1. Product Goal

`Poofer_v2` is a two-node networked flame-effect controller.

Each physical poofer has:

- one ESP32-C3 controller
- one WiSeFire interface
- one first WS2812 pixel on the ESP32 used as the local status LED

The operator uses one browser-based control console to:

- fire `stage-left`
- fire `stage-right`
- fire `both`

The system must support:

- independent firing
- best-effort synchronized firing
- variable hold times from short taps to multi-second holds
- low cognitive load during operation
- fail-closed safety behavior

The system does not attempt strict deterministic distributed synchronization in v2.

## 2. Hard Scope Limits

These are explicit v2 constraints:

- Exactly two poofer nodes are supported.
- The only valid operator-facing roles are:
  - `stage-left`
  - `stage-right`
  - `unassigned`
- There is exactly one built-in group target:
  - `both`
- Exactly one browser/client controls the show at a time.
- A dedicated external Wi-Fi network is required.
- `LoRa` is out of scope for v2.
- `LLDP` is out of scope for v2.
- Browser-to-node control is direct; there is no central broker or coordinator process in v2.

## 3. Non-Goals

The following are explicitly not required in v2:

- support for more than two poofers
- arbitrary named groups
- multi-browser shared control
- distributed consensus or leader election
- sub-millisecond timing guarantees
- operation without an external Wi-Fi network
- cloud services

## 4. System Topology

The topology is:

- one external Wi-Fi AP/router
- one browser control console on that network
- one `stage-left` poofer node on that network
- one `stage-right` poofer node on that network

Each poofer node:

- joins the external Wi-Fi network as a client
- exposes an HTTP server for node-local pages if needed
- exposes a websocket endpoint for control and telemetry
- advertises itself over `mDNS`
- persists its assigned role locally

The browser:

- discovers nodes over `mDNS`
- opens one websocket connection to each discovered node it intends to control
- renders one live operator UI
- owns the global `Armed` gate

No poofer node hosts the network.

## 5. Core Safety Tenets

These statements are normative:

- Better to miss a poof than to poof when not intended.
- A stale command must never produce a poof.
- Loss of control liveness must resolve to stop, not continued firing.
- Local node safety is authoritative for physical actuation.
- The browser may request fire; the node decides whether it is allowed to execute.
- Setup and configuration modes must be non-firing modes.

## 6. Identity Model

Each node has two identities:

- `hardware_id`
  - stable machine identity
  - derived from immutable hardware information such as MAC or chip identifier
  - never changes during normal operation
- `role`
  - operator-facing deployment role
  - one of `stage-left`, `stage-right`, `unassigned`
  - persisted locally on the node

The browser must treat `hardware_id` as the true identity.

The role is an assignment on top of the stable identity.

## 7. Role Assignment and Persistence

Each node persists its current role in non-volatile storage.

Required properties:

- Role survives reboot and power loss.
- Role can be changed by the browser setup flow.
- Node may boot with `unassigned`.
- Browser must detect invalid deployment states:
  - missing `stage-left`
  - missing `stage-right`
  - duplicate `stage-left`
  - duplicate `stage-right`
  - multiple nodes with conflicting role claims

When browser detects an invalid deployment state, live firing must be inhibited and the browser must enter the setup UX.

## 8. Discovery

Primary discovery mechanism is `mDNS`.

Each node must advertise:

- a service type identifying it as a `Poofer_v2` node
- a reachable hostname or address
- enough metadata for the browser to identify the node

Minimum discovery data required by the browser:

- `hardware_id`
- `role`
- network endpoint information sufficient to connect by websocket

The browser may cache known nodes for convenience, but `mDNS` is the source of truth for live discovery in v2.

Manual fallback for discovery may be added later, but it is not required for the first implementation.

## 9. Ownership Model

There is one active browser controller at a time.

Each node must maintain the concept of one active control session.

Required behavior:

- Node accepts fire-control commands only from the currently active session.
- Node must reject fire-control commands that do not belong to the active session.
- Browser must establish session ownership before issuing fire commands.
- Browser reconnect may require session reacquisition.

The exact session-claim handshake may be implementation-defined, but the following semantics are required:

- there is a controller identity
- there is a node-side notion of active owner
- ownership ambiguity inhibits firing

## 10. Fire Control Model

Fire control is not a single event. It is a live hold-driven interaction.

The logical interaction for any target is:

1. `DOWN`
2. repeated `HOLD`
3. `UP`

This model must support:

- very short taps
- medium holds
- long holds up to the configured max

Fire duration is determined by continued valid liveness of the control path, not by blindly trusting one up-front duration number.

## 11. Targets

Valid command targets are:

- `stage-left`
- `stage-right`
- `both`

Semantics:

- `stage-left` addresses only the node with role `stage-left`
- `stage-right` addresses only the node with role `stage-right`
- `both` is one logical operator target that fans out to both role nodes

The browser UI must present these as three direct fire controls.

## 12. Best-Effort Group Semantics

`both` is best-effort, not all-or-nothing.

If the browser issues a valid `both` fire interaction:

- any targeted node that is online, owned by the current controller, ready, armed at the browser level, and locally willing to fire may execute
- any targeted node that is offline, not owned, unready, faulted, stale, or otherwise uncertain must not execute

Therefore:

- if `stage-left` is absent and `stage-right` is available, `stage-right` may still fire when target is `both`
- if both are available, both may fire
- if neither is available, neither fires

This is intentional.

## 13. Timing Goal

V2 synchronization goal is perceptual, not instrument-grade.

Normative target:

- Firing of `both` should appear simultaneous to a human audience.
- The design target is to keep visible skew under `100 ms` on a healthy local network.

V2 uses immediate best-effort execution, not scheduled future timestamps.

However, the protocol should be structured so a future `execute_at` field could be added later without redesigning the whole message family.

## 14. Local Node State Machine

Each node owns its own runtime and safety state.

At minimum, node behavior must distinguish these conditions:

- `disconnected` from the browser control session
- `ready`
- `firing`
- `faulted` or `inhibited`
- `unassigned`

Implementation may use a formal enum plus flags, but the observable semantics must support:

- Node can be connected but not fireable.
- Node can be disconnected.
- Node can be firing only while valid control liveness continues.
- Node can stop independently of the other node.

The node must never depend on the other poofer's state to remain safe.

## 15. Browser Control State

The browser owns global operator state including:

- whether the UI is in normal control mode or setup mode
- whether `Armed` is true or false
- current node discovery results
- per-node displayed status
- active hold interaction, if any

The browser does not own the truth of whether a node is physically safe to fire.

The browser must treat node telemetry as authoritative for node-local readiness and actual execution state.

## 16. Armed Gate

The browser has a global `Armed` control.

Normative semantics:

- Default on page load, reload, or fresh browser session is `Armed = false`.
- No outgoing fire commands may be initiated when `Armed = false`.
- If operator changes from armed to disarmed during an active hold, the browser must immediately terminate that hold interaction by sending the appropriate release/stop semantics to all currently targeted nodes.
- Browser remaining connected while disarmed is allowed and expected.
- Telemetry and setup remain available while disarmed.

Armed is a browser-side command gate, not a replacement for node-local safety logic.

## 17. Setup Mode

The browser contains a dedicated `Poofer ID Setup` UX.

Its purpose is:

- initial assignment
- reassignment
- conflict resolution

Setup mode must be entered when:

- operator explicitly requests it
- any node is `unassigned`
- required role is missing
- duplicate role assignment is detected
- deployment validity is otherwise ambiguous

Setup mode semantics are mandatory:

- Entering setup mode forces `Armed = false`.
- Firing controls are disabled while setup mode is active.
- Exiting setup mode does not restore a previous armed state.
- Operator must re-arm intentionally after returning to normal mode.

The setup UX must allow:

- viewing discovered nodes by `hardware_id`
- seeing each node's current stored role
- assigning role `stage-left`
- assigning role `stage-right`
- assigning role `unassigned`
- persisting the new role to the node

## 18. UI Requirements

The operator UI is intentionally simple.

Primary live controls:

- one `1x1` fire button for `stage-left`
- one `2x1` fire button for `both`
- one `1x1` fire button for `stage-right`
- one global `Armed` slider
- one entry point into `Poofer ID Setup`

The UI must preserve the v1 three-second depletion gauge concept for fire buttons.

Button behavior:

- hold interaction depletes the gauge while actively firing
- gauge refills after release
- left and right buttons display per-node `last_hold_ms`
- `both` does not synthesize a fake shared `last_hold_ms`

The operator must infer combined outcomes from per-node status rather than a fabricated aggregate duration.

## 19. UI Visual State and Color Mapping

Color meanings are inherited from v1:

- `green` = ready / good-to-go
- `blue` = disconnected
- `red` = connected but unavailable, faulted, inhibited, or otherwise not willing to fire
- `orange` = actively firing

These colors must match between:

- the browser button state for a node
- the corresponding node's first physical WS2812 pixel

`Armed = false` must not consume a separate status color.

Instead, when disarmed:

- fire buttons are muted
- fire buttons are visually blurred or otherwise clearly de-emphasized
- status colors remain semantically meaningful underneath that treatment

This separation is intentional:

- color communicates node state
- the armed slider communicates whether outgoing poof commands are allowed

## 20. WS2812 Status LED Requirements

Each node's first WS2812 pixel is the node-status indicator.

Normative behavior:

- It must reflect the same semantic color state the browser shows for that node.
- During active firing, `orange` overrides ready coloring.
- Fault/inhibit conditions must show `red`.
- Lack of control connection or disconnected state must show `blue`.

Implementation details of other pixels are outside the scope of this document except that local actuation behavior must remain safe.

## 21. Protocol Requirements

The wire protocol may be text or JSON, but JSON messages are recommended for v2 for clarity and extensibility.

At minimum, protocol must represent:

- controller/session claim
- node identity
- node role
- target identity
- command type
- command ID
- command expiry or validity limit
- telemetry/status
- role assignment

Every fire-control message must include:

- `command_id`
- `target`
- `type`
- session identity

Recommended message families:

- session claim / session accepted / session rejected
- command `DOWN`
- command `HOLD`
- command `UP`
- telemetry `status`
- command acknowledgment
- role assignment request / response

## 22. Fire Command Semantics

Required semantics for fire commands:

- `DOWN`
  - requests start of fire interaction for the target
- `HOLD`
  - extends liveness of an already active fire interaction
- `UP`
  - requests end of fire interaction for the target

Each node must enforce:

- `DOWN` is ignored or rejected when local prerequisites are not satisfied
- `HOLD` is ignored unless a matching active fire interaction exists
- `UP` is safe to process even if local fire interaction is already absent

The system should tolerate duplicate `UP` without dangerous behavior.

## 23. Hold Liveness

Node-local liveness timeout is mandatory.

Required behavior:

- Node starts a local hold-liveness timer when a fire interaction begins.
- Each valid `HOLD` refreshes that timer.
- Expiry of the timer forces local stop.
- Loss of websocket/session causing missed `HOLD` messages must therefore stop fire.

Exact timing values may be tuned during implementation, but v1-style short liveness windows are the model.

## 24. Command Validity and Deduplication

Each node must protect against stale or duplicate commands.

Required behaviors:

- Track recently seen `command_id` values for the active session.
- Ignore or reject expired commands.
- Ignore or reject duplicate `DOWN` that would replay stale intent.
- Treat uncertain command validity as non-fireable.

No stale command may start or prolong a poof.

## 25. Node Telemetry

Each node must publish enough status for the browser to render the live UI honestly.

Minimum node telemetry fields:

- `hardware_id`
- `role`
- `connected` or session-owned indicator
- `ready`
- `firing`
- `faulted` or `inhibited`
- `last_hold_ms`
- optional `reason` for not-ready or faulted state

Telemetry should allow the browser to distinguish:

- unreachable node
- reachable but unassigned node
- reachable but not owned node
- reachable but inhibited/faulted node
- reachable and ready node
- actively firing node

## 26. Browser Fan-Out Behavior

When operator presses a button:

- `stage-left`
  - browser sends one logical fire interaction only to the node assigned `stage-left`
- `stage-right`
  - browser sends one logical fire interaction only to the node assigned `stage-right`
- `both`
  - browser creates one logical target intent and fans it out to both active node connections

The browser should keep one logical operator interaction for `both`, but track per-node outcomes underneath it.

This allows future extension without changing the operator mental model.

## 27. Result Reporting

The UI should not fabricate aggregate numbers.

Required reporting model:

- per-node status is primary
- per-node `last_hold_ms` is shown on left/right controls
- `both` should not display a fake unified duration

The browser may later display informational summaries such as partial execution, but the first implementation can rely on per-node indicators.

## 28. Fault and Inhibit Model

`Blue` and `red` have different meanings.

- `blue`
  - browser cannot currently communicate with the node
  - or node is not present on the control path
- `red`
  - browser can communicate with the node
  - but the node is not willing to fire

Examples of `red` conditions include:

- role conflict or invalid setup state
- local safety inhibit
- stale or invalid control session
- malformed or expired fire command
- dead-man timeout recovery state
- internal fault condition

The exact set of fault reasons may evolve, but the semantic distinction between `blue` and `red` is required.

## 29. Browser Disconnect and Reload Behavior

Safety behavior on browser-side interruption is mandatory.

Required behavior:

- Browser reload starts disarmed.
- Browser disconnect causes nodes to age out active hold interactions and stop locally.
- Reconnection does not implicitly re-arm or resume firing.
- Ownership/session may need to be reacquired before the browser can fire again.

## 30. Node Reboot Behavior

On node reboot:

- Persisted role must be retained.
- Node must return to a non-firing state.
- Node must rejoin the Wi-Fi network and re-advertise over `mDNS`.
- Browser must rediscover or reconnect before node can participate again.

Reboot must never resume a prior active fire interaction.

## 31. Suggested Initial JSON Shapes

These are recommended shapes, not the only possible encoding.

Session claim:

```json
{
  "type": "claim",
  "controller_id": "browser-uuid",
  "session_id": "session-uuid",
  "sent_at_ms": 1234567890
}
```

Node status:

```json
{
  "type": "status",
  "hardware_id": "esp32c3-abc123",
  "role": "stage-left",
  "owned": true,
  "ready": true,
  "firing": false,
  "faulted": false,
  "reason": null,
  "last_hold_ms": 250
}
```

Fire command:

```json
{
  "type": "fire",
  "phase": "DOWN",
  "command_id": "cmd-uuid",
  "controller_id": "browser-uuid",
  "session_id": "session-uuid",
  "target": "stage-left",
  "expires_in_ms": 150,
  "sent_at_ms": 1234567890
}
```

Role assignment:

```json
{
  "type": "assign_role",
  "controller_id": "browser-uuid",
  "session_id": "session-uuid",
  "hardware_id": "esp32c3-abc123",
  "role": "stage-left"
}
```

Acknowledgment:

```json
{
  "type": "ack",
  "command_id": "cmd-uuid",
  "hardware_id": "esp32c3-abc123",
  "phase": "DOWN",
  "result": "accepted"
}
```

## 32. Implementation Milestones

Recommended build order:

1. Refactor single-node firmware into a role-aware node firmware with persistent role and updated status model.
2. Preserve safe local hold-based firing and status LED behavior on each node.
3. Add direct websocket session ownership and JSON protocol support.
   Circle back: finish true session ownership semantics and controller claim behavior before treating the distributed control plane as complete.
4. Add `mDNS` discovery metadata needed for browser setup.
5. Build browser setup UX for role assignment and conflict handling.
6. Build live control UI with `Armed` slider and left/both/right buttons.
7. Implement browser fan-out for `both`.
8. Verify safety edge cases:
   - browser reload
   - node disconnect
   - duplicate role
   - unassigned node
   - partial availability during `both`

## 33. Acceptance Criteria

The first acceptable v2 implementation satisfies all of the following:

- Two nodes can be discovered on a dedicated Wi-Fi network.
- Roles can be assigned and persisted as `stage-left` and `stage-right`.
- Setup mode forces disarm and disables firing.
- Role conflicts block live control and redirect to setup mode.
- Browser live UI exposes `stage-left`, `both`, and `stage-right`.
- Browser defaults to disarmed on load.
- Left and right nodes can fire independently.
- `both` fans out best-effort.
- A node dropping off the network does not cause an unintended poof.
- A browser disconnect or missed hold liveness stops any active poof.
- Browser button colors and node WS2812 status colors remain semantically aligned.
