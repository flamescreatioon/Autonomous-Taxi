# Diagnostics — the telemetry sample and the display contract


> **Status (2026-10-28).** The invariants below are the rules the implementation must satisfy.
> The **enforcement tests named here do not exist yet** — `tests/test_safety_invariants.py`,
> `tests/test_protocol_freshness.py` and the other files referenced are to be written along with
> the v2 modules. The rules themselves are binding regardless of when the tests arrive.

---

## The sample shape

Every telemetry value carries four fields. All four are required.

```python
@dataclass(frozen=True)
class TelemetrySample:
    source: str      # "simulated" or the device identity
    age_s: float     # how old the sample is
    valid: bool      # may this sample be used?
    values: dict     # the readings — EMPTY when invalid
    reason: str | None  # why it is invalid, when it is
```

### `source`

Never blank, never implied. `"simulated"` for the simulation-backed source; the device identity
for a real board.

This is the field that stops a simulated reading from being mistaken for a measurement. The
diagnostics screen must show it, and must style simulated values differently from measured ones.

### `age_s`

How old the sample is. **`age_s = 0` does not mean healthy** — it means the sample was just
received, which says nothing about whether the underlying sensor is producing valid data. A
firmware that has never had a UART line reports `NO_TELEMETRY` rather than a zero age.

### `valid`

Whether the sample may be used. `False` for stale data, missing keys, `sensors_ready=false`, a
parse failure, or a transport failure.

### `values`

**Empty when `valid` is `False`.** Never fill it with defaults, zeros or last-known-good values.

### `reason`

Why the sample is invalid. A diagnostic screen that says "unavailable" without a reason is half a
diagnostic.

---

## The health rules

`taxi/comms/health.py::assess()` is to apply them:

| Condition | Result |
|---|---|
| The sample is older than the permitted age | invalid — stale |
| A required key is missing | invalid — missing key |
| The source reports `sensors_ready=false` | invalid — sensors not ready |
| The transport failed | invalid — transport error, empty values |
| The payload was malformed | invalid — parse error |
| Otherwise | valid |

The single rule behind all of them:

> **Missing or stale sensors are invalid, never substituted with fabricated healthy values.**

---

## The two sources

### Simulation — `taxi/comms/simulated.py` (to be created)

Wraps the existing `Supervisor`. Reports `source="simulated"`. Becomes invalid when the
simulation reports a sensor failure or a link failure.

### Device — `taxi/comms/esp32.py` (to be created)

**Read-only by design.** It parses `/telemetry` and `/capture` and **cannot issue motion
commands**. This is the point of the adapter, not a limitation of it.

Bounded parsing:

| Limit | Value |
|---|---|
| `MAX_TELEMETRY_BYTES` | 2048 |
| `MAX_TELEMETRY_FIELDS` | 32 |

A bounded parser on a device link is a security property: an unbounded parser lets a malformed
or hostile payload exhaust memory on the gateway.

Failure modes, all producing `valid=False` with a reason and empty values:

- the transport times out,
- the payload is malformed,
- the payload reports `sensors_ready=false`,
- a required key is missing.

---

## The bench firmware's reported state

`firmware/arduino/src/main.cpp` reports:

```json
{"type":"telemetry","source":"bench_firmware","uptime_ms":0,
 "motion_enabled":false,"sensors_ready":false,"reason":"boot_disarmed"}
```

`sensors_ready` is `false` because the echo callbacks are empty — the sonar FSM runs but never
observes a real echo, so every reading stays invalid. This is honest and must stay visible.

### What the endpoints do and do not confirm

| Endpoint | Confirms | Does **not** confirm |
|---|---|---|
| `GET /capture` | A JPEG was produced | Anything about the image's content |
| `GET /telemetry` | The last unvalidated UART line and its age | That the sensors are working |
| `POST /stop` | That a STOP request was sent to the Arduino | That the vehicle stopped |

`NO_TELEMETRY` is reported before the first UART line arrives. The `/stop` endpoint returns 503
when UART is disabled.

---

## The display contract

Every telemetry row on the diagnostics screen shows:

| Column | Source |
|---|---|
| Reading | `values[key]` — or `unavailable` when invalid |
| Source | `source` |
| Age | `age_s` |
| Validity | `valid` — with the `reason` beside it when false |

Do not colour a value by validity alone. Pair the colour with a word (`valid` / `stale` /
`unavailable`).

### Never do this

- Render a missing sensor as `0`.
- Render a missing sensor as a plausible-looking number.
- Drop the source label because "we know it's the simulation".
- Show `age = 0` as evidence of health.
- Show a `STOP` acknowledgement as a measured braking event.

---

## Hardware status in the UI

Until hardware is confirmed and a board is connected, the diagnostics screen should show the
hardware section as **unconfirmed** with the specific missing facts, not as empty or as zeros.
See `docs/HARDWARE.md` and the checklist in the `taxi-hardware` skill.
