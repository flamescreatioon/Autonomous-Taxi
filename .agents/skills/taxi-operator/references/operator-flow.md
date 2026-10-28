# Operator flow


> **Status (2026-10-28).** The invariants below are the rules the implementation must satisfy.
> The **enforcement tests named here do not exist yet** — `tests/test_safety_invariants.py`,
> `tests/test_protocol_freshness.py` and the other files referenced are to be written along with
> the v2 modules. The rules themselves are binding regardless of when the tests arrive.

---

## Stop, rearm, reset

Three distinct operations with distinct meanings. The UI must label them distinctly, because
conflating them is how an operator comes to believe the vehicle is safe when it is not.

| Action | What it actually does | Vehicle may move afterwards? |
|---|---|---|
| **Stop** | The application stops commanding motion. The trip moves to a terminal status. | No — not until an explicit rearm. |
| **Rearm** | Clears a latched `FAULT` / `STOPPED` / `COMPLETE` state, returning the vehicle to `READY`. | Yes, if rearm is permitted. |
| **Reset** | Resets the **simulation** to stop A, clears the path and trip. | Only when not `ACTIVE` and no simulated hazards remain. |

### Stop

- The application stops. It does **not** brake.
- The UI must not say "vehicle stopped" — say "stop requested" or "motion stopped".
- A repeated stop on a latched fault does not clear it, and the UI should show that the state is
  unchanged rather than showing a fresh confirmation.

### Rearm

- `rearm()` **refuses** while any hazard remains (obstacle, sensor failure, link failure) or while
  telemetry is stale.
- When it refuses, it says why. Show that reason. Do not offer a retry that will fail identically.
- Rearm is not automatic. It is not triggered by a reconnect, a page reload or a clear-hazards
  action.

### Reset

- Refused while `ACTIVE`. Stop the trip first.
- Refused while simulated hazards remain.
- **Reset has no physical equivalent.** Do not imply the real vehicle can be teleported to stop A.

---

## One active mission

The vehicle is a single shared resource with exactly one active mission. A second concurrent
mission request is **rejected**, not queued and not silently dropped.

Enforced on the server. The UI should also reflect it — when a mission is active, the request
control is visibly unavailable rather than failing on submit.

---

## The fault display

A latched fault must be unmistakably visible: the vehicle state, the **reason**, and the fact that
it will not clear on its own.

```text
FAULT — sensor_stale
Motion is stopped and will not resume automatically.
[Re-arm]   (disabled until rearm is permitted)
```

If rearm is currently refused, either disable the button and show the reason, or let the user try
and show the server's explanation. Both are acceptable; silently doing nothing is not.

---

## Live supervision

The read-only snapshot WebSocket publishes at 5 Hz. The operator screen should show at least:

| Field | Source |
|---|---|
| Vehicle state | `snapshot.state` |
| Trip id, origin, destination, status | `snapshot.trip` |
| Speed | `snapshot.speed` |
| Current stop | `snapshot.current_stop` |
| Fault reason | `snapshot.reason` |
| Telemetry source and age | `snapshot.telemetry` |
| `motor_output_enabled` | `snapshot.motor_output_enabled` |

**`motor_output_enabled` must be displayed as `false`.** It is the constant that makes the whole
system safe to develop against. If it ever reads `true`, something is deeply wrong — surface it
prominently.

### The source label

If the telemetry is simulated, say so. A simulated value rendered identically to a measured one
is misleading. This matters more on the operator screen than anywhere else, because the operator
is the person who decides whether to trust the display.

### Disconnection

If the WebSocket drops, the operator screen must say so. An operator watching a frozen snapshot
may believe they are seeing a stationary vehicle when they are seeing a dead connection.

---

## What the operator may not do

- Grant motion directly, or bypass the command path.
- Bypass the Arduino watchdog, or override the emergency reflex.
- Resume a terminal trip.
- Reset a fault by fiat.
- Enable a physical output.

Safety authority flows downward only: the operator may *request* motion through the normal path;
only the Arduino may *grant* it. See `docs/adr/ADR-0001.md`.

---

## Splitting the dashboard

The existing `taxi/static/index.html` and `taxi/static/app.js` contain the map, the current
mission panel, the state display, the speed, the stop/rearm controls and the simulator hazard
controls.

The split into `/operator` must **preserve the working A-to-B simulation**. That is the exit gate
for the task, not a nice-to-have.

After the split:
- The original path may redirect to `/operator` or remain as the operator page — either is fine, as
  long as one works.
- Shared code should move to `taxi/static/api-client.js` and `taxi/static/shared.css` (both
  planned, both Member 2's coordination task). Do not leave three near-duplicate copies of the
  fetch logic across the four pages.

---

## Design tokens

| Role | Value |
|---|---|
| Background | `#080D0A` |
| Surface | `#122019` |
| Primary green | `#22C55E` |
| Highlight yellow | `#FACC15` |
| Text | `#F5F7F5` |
| Muted | `#A7B5AC` |

Black text on bright green and yellow. **Pair every status colour with a word** — colour alone is
not an accessible signal, and the dashboard must remain readable in a monochrome browser, in a
terminal, or in a log.

---

## API surface

See `taxi/api.py` for the definitive definitions and `/docs` for the OpenAPI view.

| Concern | Where |
|---|---|
| Trip request | `POST /api/trips` |
| Trip status | `GET /api/trips/{id}` |
| Stop | the operator stop endpoint |
| Rearm | the operator rearm endpoint |
| Simulation reset | the reset endpoint |
| Hazard injection | the hazard endpoints |
| Live snapshots | the read-only WebSocket at 5 Hz |

All mutation and history endpoints require the bearer token. Public status, the stop definitions,
the API documentation and the read-only WebSocket are appropriate only for the loopback
development instance.
