# Core invariants and the tests that enforce them


> **Status (2026-10-28).** The invariants below are the rules the implementation must satisfy.
> The **enforcement tests named here do not exist yet** — `tests/test_safety_invariants.py`,
> `tests/test_protocol_freshness.py` and the other files referenced are to be written along with
> the v2 modules. The rules themselves are binding regardless of when the tests arrive.

Every invariant below is checked by an automated test. If a change appears to require breaking
one, the change is wrong — fix the change, not the test.

---

## 1. Terminal trip statuses never revert

**Where:** `taxi/mission/state.py::TERMINAL_TRIP_STATUSES`, `taxi/mission/state.py::is_terminal()`

`COMPLETED`, `CANCELLED`, `FAULTED` and `INTERRUPTED` are final. There is no code path that takes
a terminal trip back to `ACTIVE`. A server restart that finds an active trip marks it
`INTERRUPTED` rather than resuming it.

**Enforced by:** `tests/test_supervisor.py` (the interruption test). `tests/test_mission_state.py` is to be written.

---

## 2. Faults latch

**Where:** `taxi/supervisor.py::stop()`, `Supervisor.rearm()`

Once `FAULT` is reached, clearing the *input* that caused it (the obstacle flag, the sensor
failure, the link failure) does **not** clear the state. Only an explicit `rearm()` returns the
vehicle to `READY`, and `rearm()` itself refuses while any hazard remains or while telemetry is
stale.

A repeated stop request cannot clear a latched fault either.

**Enforced by:** `tests/test_supervisor.py` (fault latching and rearm tests).

---

## 3. A reconnect never authorises motion

**Where:** `taxi/protocol/motion.py::is_fresh()`, `firmware/arduino/src/watchdog.cpp`

Three independent mechanisms:

- The Arduino watchdog starts **expired** after boot or reset. The first accepted command rearms
  it. A board that has just rebooted cannot move until a fresh command arrives.
- A motion command carries a `session_id`. A session change flushes any pending commands, so a
  command queued before a reconnect is discarded on arrival.
- A heartbeat is not a motion command and carries no velocity. Only an accepted, fresh,
  correctly-sequenced motion command advances the watchdog.

**Enforced by (to be written):** `tests/test_protocol_freshness.py`, `tests/test_safety_invariants.py`.

---

## 4. One server process owns one supervisor

**Where:** `taxi/api.py` (the FastAPI lifespan)

The lifespan creates exactly one `Store` and one `Supervisor`. Do not add API workers. A second
supervisor means two sources of truth about the vehicle's state, and they will disagree.

Request handlers must not `await` while mutating the core, so mutations stay serialised on the
event loop. SQLite writes are synchronous and appropriate at this scale; a storage error faults
motion in memory.

**Enforced by:** `tests/test_api.py` (the live simulation check), `tests/test_supervisor.py`.

---

## 5. Missing or stale sensors are invalid, never fabricated

**Where:** `taxi/comms/health.py::assess()`, `taxi/planning/occupancy.py::SensorReading`,
`taxi/comms/esp32.py`, `taxi/perception/pipeline.py`

- A `SensorReading` with `distance_cm=None` marks **nothing** — neither free nor occupied.
- A telemetry sample that is stale, missing a required key, or reporting `sensors_ready=false` is
  `valid=False`, with a reason and empty values.
- A frame older than `max_age_s` is discarded, not delivered.
- A device adapter that fails to parse or times out yields an invalid sample, never a default.

**Enforced by (to be written):** `tests/test_comms.py`, `tests/test_planning.py`,
`tests/test_perception_queue.py`.

---

## 6. The host cannot enable a motor

**Where:** `taxi/supervisor.py` (`motor_output_enabled`), all of `taxi/`

`motor_output_enabled` is hardcoded `False` and the `snapshot()` reports it. No module in `taxi/`
may contain an actuator write path.

**Enforced by (to be written):** `tests/test_safety_invariants.py`, which scans every `taxi.*` module's source for
patterns that could enable a motor output.

---

## 7. Application stop ≠ physical emergency stop

**Where:** `taxi/api.py` (the stop endpoint), `firmware/arduino/src/main.cpp`

`stop()` and the operator stop endpoint mean "the application will not command motion". They do
**not** mean "the vehicle is physically stationary" — the software does not know that, and the
bench firmware has no actuator and cannot brake.

The physical emergency stop is independent hardware that no protocol can inhibit. It is not
implemented in firmware, and it is not represented by the application's stop button.

**Enforced by (to be written):** `tests/test_safety_invariants.py` (the firmware pins stay unset), and by review.

---

## 8. The firmware cannot drive anything

**Where:** `firmware/arduino/src/main.cpp`, `firmware/esp32_s3/src/main.cpp`

- The Arduino `main.cpp` contains no `analogWrite`, no `digitalWrite`, no `MOTOR_PIN`.
- The echo callbacks `onTrigger` and `onCapture` are empty, so every sonar reading stays invalid.
- Every `kCameraPin*`, `kUartTxPin` and `kUartRxPin` in the ESP32-S3 `main.cpp` is `-1`.

**Enforced by (to be written):** `tests/test_safety_invariants.py`.

---

## 9. The arbiter tier order is identical in Python and C++

**Where:** `taxi/mission/safety.py::TIER_ORDER`, `firmware/arduino/src/arbiter.h`

The order is:

```text
TIER_FAIL_SAFE  <  TIER_EMERGENCY_REFLEX  <  TIER_NOMINAL
```

The test reads the tier names and their order directly out of the C++ header and compares them to
the Python side, so the two cannot drift apart silently.

**Enforced by (to be written):** `tests/test_safety_invariants.py`.

---

## 10. A missing sensor does not trigger the emergency reflex

The reflex fires only on a **valid** measurement below threshold. "No reading" is not "reading
zero". A missing sensor therefore falls through to the fail-safe tier rather than being mistaken
for an imminent collision.

**Enforced by (to be written):** `tests/test_safety_invariants.py`, `tests/test_planning.py`.
