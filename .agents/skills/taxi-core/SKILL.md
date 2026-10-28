---
name: taxi-core
description: Use this skill whenever work touches the Autonomous Taxi's mission core, supervisor, state machine, trip statuses, faults, latching, rearm, snapshots, telemetry shape, or the simulation's safety invariants. Trigger on mentions of supervisor.py, mission/, trip status, vehicle state, READY/ACTIVE/COMPLETE/STOPPED/FAULT, latched fault, rearm, terminal status, CANCELLED/COMPLETED/FAULTED/INTERRUPTED, snapshot, one supervisor per process, stale telemetry, "stale means invalid", motion request, motor_output_enabled, or any change to taxi/api.py that could alter the trip lifecycle. Also read this before adding a new API endpoint that mutates a trip, and before changing anything under taxi/mission/, taxi/protocol/ or taxi/comms/.
---

# Taxi Core — mission state, safety invariants and the simulation contract

> **Repository state (2026-10-28).** The three-tier architecture these skills describe is
> **specified but not implemented**. There is no `taxi/protocol/`, `taxi/comms/`,
> `taxi/perception/`, `taxi/planning/` or `taxi/mission/` package, no `firmware/esp32_s3/`
> target, and no binary framing code on either board. What exists is the v0.1 simulation, the
> ESP32-CAM bench starter and the Arduino newline-ASCII bench parser. Treat every path and
> constant named below as **the target design**, not as code you can read or run. See
> `docs/VALIDATION.md` for the verified state of the current tree.


The core is small and deliberately rigid. Almost every change here can silently weaken a safety
property, so read this file before editing, not after.

## The non-negotiable invariants

These are enforced by the existing test suite (`tests/test_api.py`, `tests/test_supervisor.py`)
and, once the v2 tiers land, by `tests/test_safety_invariants.py`. If a change
appears to require breaking one, the change is wrong.

1. **Terminal trip statuses never revert.** `COMPLETED`, `CANCELLED`, `FAULTED` and `INTERRUPTED`
   are final. `taxi/mission/state.py::TERMINAL_TRIP_STATUSES` is to become the single source of
   truth; today the terminal statuses live inline in `taxi/supervisor.py`.
   There is no "resume a terminal trip" path.

2. **Faults latch.** Once `FAULT` is reached, clearing the *input* that caused it does not clear the
   state. Only an explicit `rearm()` returns the vehicle to `READY`, and `rearm()` itself refuses
   while hazards remain or telemetry is stale.

3. **A reconnect never authorises motion.** Re-establishing authority is always deliberate. The
   Arduino watchdog starts *expired*; the host's `rearm()` is a separate, explicit act.

4. **One server process owns one supervisor.** Do not add API workers, do not start a second
   supervisor in a test or a script. `taxi/api.py` creates one Store and one Supervisor in the
   FastAPI lifespan. If you need a second supervisor in a test, construct one directly and keep it
   out of the app.

5. **Missing or stale sensors are invalid, never substituted with fabricated healthy values.** This
   applies to `taxi/comms/health.py::assess()`, to `taxi/planning/occupancy.py::SensorReading`
   (a `None` distance marks *nothing* — neither free nor occupied), and to every telemetry adapter.
   An invalid sample carries a reason and empty values.

6. **The host cannot enable a motor.** `motor_output_enabled` is `False` and stays `False`. No
   module in `taxi/` may contain an actuator write. The safety test scans the source and fails if
   one appears.

7. **Application stop ≠ physical emergency stop.** `stop()` and the operator stop endpoint mean
   "the application will not command motion". They do not mean "the vehicle is stationary". The
   physical emergency stop is independent hardware that no protocol can inhibit, and the bench
   firmware has no implementation of it.

## The vocabulary

`taxi/mission/state.py` (to be created) is to be the canonical source. Use the enums; do not
retype the string literals.

**Vehicle state** — what the vehicle is doing:

| State | Meaning |
|---|---|
| `READY` | No movement. Permits a mission only at a named stop with healthy inputs. |
| `ACTIVE` | Exactly one trip owns the vehicle. |
| `COMPLETE` | Destination reached, speed zero, trip `COMPLETED`. |
| `STOPPED` | Operator stop or cancel, speed zero, trip `CANCELLED`. |
| `FAULT` | Latched fault: obstacle, stale sensor, lost link, timing, or supervisor exception. |

**Trip status** — what happened to a trip:

| Status | Terminal? |
|---|---|
| `ACTIVE` | no |
| `COMPLETED` | yes |
| `CANCELLED` | yes |
| `FAULTED` | yes |
| `INTERRUPTED` | yes — used when a server restart interrupts an active trip |

Trip status and vehicle state are deliberately separate. A trip can be `COMPLETED` while the
vehicle is `COMPLETE`, and the vehicle can be `FAULT` with no trip at all.

## Where things live

| Path | Role |
|---|---|
| `taxi/supervisor.py` | The v0.1 simulation. Public surface is stable: `Supervisor`, `Conflict`, the state strings, `snapshot()`. |
| `taxi/mission/state.py` | **To be created.** `VehicleState`, `TripStatus`, `TERMINAL_TRIP_STATUSES`, `is_terminal()`. Today these strings live inline in `taxi/supervisor.py`. |
| `taxi/mission/safety.py` | **To be created.** `SubsumptionArbiter` — the host-side mirror of the Arduino arbiter. |
| `taxi/api.py` | FastAPI routes and the lifespan that owns the single supervisor. |
| `taxi/storage.py` | SQLite trips and events. Synchronous; suitable only at prototype scale. |
| `taxi/navigation.py` | The route graph and `route()`. |

## Working rules

- **Do not change `Supervisor`'s public surface.** `snapshot()`'s shape, the state strings and the
  `Conflict` exception are consumed by the dashboard and by the existing tests. Add alongside;
  do not rename.
- **Extract, do not rewrite.** When moving logic out of `supervisor.py`, keep behaviour identical
  and let the existing tests prove it. `taxi/mission/state.py` is to be extracted this way.
- **New endpoints that mutate a trip must go through the supervisor.** Do not write to the store
  directly from a route handler; that bypasses latching and terminal-status checks.
- **Never resume.** There is no code path that takes a terminal trip back to `ACTIVE`. If a server
  restart finds an active trip, it marks it `INTERRUPTED`.

## Verification

After any change under `taxi/`, run from the repo root:

```bash
./.venv/Scripts/python.exe -m pytest -q
./.venv/Scripts/ruff.exe check .
```

If you changed `taxi/static/app.js` or any other browser JavaScript:

```bash
node --check taxi/static/app.js
```

(`ruff` and `python` are not on `PATH` in this environment — use the project venv, or `py`.)

## Reference

- `references/invariants.md` — the invariants in checklist form, with the test that enforces each.
- `references/ownership.md` — who owns which file, so parallel work does not collide.
- `docs/ARCHITECTURE-v2.md` — the three-tier target architecture.
- `docs/adr/` — the architecture decision records.
