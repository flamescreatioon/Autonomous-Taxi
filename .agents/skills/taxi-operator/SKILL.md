---
name: taxi-operator
description: Use this skill for any work on the Autonomous Taxi's operator experience - mission control, live supervision, the operator dashboard, the /operator page, operator.js, operator.html, taxi/routers/operator.py, stop and rearm controls, mission state display, one-active-mission enforcement, fault handling from the operator's view, shared API wiring, shared route registration, shared UI tokens, api-client.js, shared.css, or coordinating work between the four team members. Trigger when someone asks how an operator watches, supervises, stops, rearms, or resets the vehicle, or when changing anything that more than one of the four interfaces depends on.
---

# Taxi Operator — supervision, stop semantics and shared contracts

> **Repository state (2026-10-28).** The three-tier architecture these skills describe is
> **specified but not implemented**. There is no `taxi/protocol/`, `taxi/comms/`,
> `taxi/perception/`, `taxi/planning/` or `taxi/mission/` package, no `firmware/esp32_s3/`
> target, and no binary framing code on either board. What exists is the v0.1 simulation, the
> ESP32-CAM bench starter and the Arduino newline-ASCII bench parser. Treat every path and
> constant named below as **the target design**, not as code you can read or run. See
> `docs/VALIDATION.md` for the verified state of the current tree.


The operator is the human in the loop for a vehicle that is not yet safe to run autonomously. The
operator interface therefore has a bias: it shows the **state of the system honestly**, including
the parts that are degraded, missing, or simulated.

## Stop, rearm and reset are three different things

| Action | Meaning | May the vehicle move afterwards? |
|---|---|---|
| **Stop** | The application will not command motion. | Not until an explicit rearm. |
| **Rearm** | Clear a latched fault / stopped state, return to `READY`. | Yes — but only if hazards are clear and telemetry is fresh. |
| **Reset** | Reset the *simulation* to stop A. | Only when not `ACTIVE` and no simulated hazards. |

These are not interchangeable and the UI must label them differently.

- **Stop is not braking.** It means the application stopped commanding. The UI must not claim the
  vehicle physically stopped, because the software does not know that.
- **Rearm is refused while hazards remain or telemetry is stale.** The `Conflict` message says why.
  Show that reason; do not offer a retry that will fail identically.
- **Reset is refused while `ACTIVE`.** Stop the trip first.
- **Reset has no physical equivalent.** Do not imply the real vehicle can be teleported to stop A.

## One active mission

The vehicle is a single shared resource with **one** active mission. Enforce this in the server.
A second concurrent mission request must be rejected, not queued and not silently dropped.

## Faults latch — and the operator must be able to see that

Once `FAULT` is reached, clearing the *input* that caused it does not clear the state. The UI must
show the vehicle as faulted and the reason, and offer rearm only when rearm is actually permitted.
Repeated stop requests cannot clear a latched fault, and the interface must reflect that a second
stop did not change anything.

## Operator controls never bypass the Arduino safety layer

The operator may request motion through the normal command path. The operator may **not**:

- grant motion directly,
- bypass the watchdog,
- override the emergency reflex,
- resume a terminal trip,
- reset a fault by fiat.

Safety authority flows downward only. See `docs/ARCHITECTURE-v2.md` and
`docs/adr/ADR-0001.md`.

## One server process, one supervisor

Do not add API workers. The FastAPI lifespan creates one Store and one Supervisor, and a second
supervisor means two sources of truth about the vehicle's state.

Request handlers must not `await` while mutating the core, so mutations stay serialised on the
event loop. SQLite writes are synchronous and fine at this scale; a storage error faults motion in
memory.

## Coordinated ownership — Member 2's job

Member 2 is the integration lead and coordinates the shared files. **Do not edit these without
coordinating:**

| Shared file | Rule |
|---|---|
| `taxi/api.py` | Route registration is coordinated. Four people editing it independently is how the API breaks. |
| `taxi/storage.py` | Schema changes are coordinated with Member 3. |
| `taxi/static/shared.css` | Shared design tokens. |
| `taxi/static/api-client.js` | Shared fetch/WebSocket client. |

**Proposed shared files** `taxi/static/shared.css`, `taxi/static/api-client.js` and shared
navigation **do not exist yet**. Creating them is Member 2's coordination task — create them once,
properly, rather than four times.

Prefer a small router module per workstream over growing `api.py`.

## Design tokens

| Role | Value |
|---|---|
| Background | `#080D0A` |
| Surface | `#122019` |
| Primary green | `#22C55E` |
| Highlight yellow | `#FACC15` |
| Text | `#F5F7F5` |
| Muted | `#A7B5AC` |

Use black text on bright green and yellow. **Pair every status colour with a word** — colour alone
is not an accessible or unambiguous signal, and the dashboard must remain readable when a status is
rendered in a terminal, a log, or a monochrome browser.

## Existing endpoints to reuse

`taxi/api.py` provides the mission-control surface: trip request, trip status, stop, rearm,
simulation reset, hazard injection, read-only snapshots and a 5 Hz read-only WebSocket. Splitting
the existing dashboard into `/operator` must **preserve the working A-to-B simulation** — that is
the exit gate for the task.

## Planned file ownership

| Path | Owner |
|---|---|
| `taxi/static/operator.html` | Member 2 |
| `taxi/static/operator.js` | Member 2 |
| `taxi/routers/operator.py` | Member 2 |
| shared route registration in `taxi/api.py` | Member 2 (coordinated) |

## Verification

```bash
./.venv/Scripts/python.exe -m pytest -q
./.venv/Scripts/ruff.exe check .
node --check taxi/static/operator.js    # if you create it
node --check taxi/static/app.js          # if you touch the existing dashboard
```

Demonstrate the A-to-B simulation still works after splitting the dashboard. See
`docs/TEAM-WORKSPACE.md`.

## Reference

- `references/operator-flow.md` — stop/rearm/reset semantics, endpoint contract and dashboard states.
- `references/ownership.md` — the file-ownership map for all four workstreams.
- `docs/TEAM-WORKSPACE.md` — Member 2's task list.
