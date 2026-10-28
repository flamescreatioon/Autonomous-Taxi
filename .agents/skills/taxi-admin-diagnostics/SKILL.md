---
name: taxi-admin-diagnostics
description: Use this skill for any work on the Autonomous Taxi's admin and diagnostics surfaces - accounts, roles, permissions, authentication, sessions, password hashing, trip ownership enforcement, SQLite schema and migrations, trip and event records, reporting, pagination, backups, admin UI, /admin page, /diagnostics page, source labels, sensor age, validity, camera state, connection state, telemetry parsing, protocol timeouts, hardware telemetry, read-only adapters, or anything touching taxi/comms/. Trigger before showing any sensor value to a user, before adding a schema migration, before implementing an account system, and whenever someone asks what the vehicle currently sees.
---

# Taxi Admin & Diagnostics — honest data, honest access

> **Repository state (2026-10-28).** The three-tier architecture these skills describe is
> **specified but not implemented**. There is no `taxi/protocol/`, `taxi/comms/`,
> `taxi/perception/`, `taxi/planning/` or `taxi/mission/` package, no `firmware/esp32_s3/`
> target, and no binary framing code on either board. What exists is the v0.1 simulation, the
> ESP32-CAM bench starter and the Arduino newline-ASCII bench parser. Treat every path and
> constant named below as **the target design**, not as code you can read or run. See
> `docs/VALIDATION.md` for the verified state of the current tree.


Two surfaces, one shared principle: **show what is actually true, and show only what the caller is
allowed to see.** A dashboard that hides degradation is worse than no dashboard, because it
converts an unknown into a false reassurance.

---

# Part 1 — Admin: accounts, roles and records

## Roles

Three roles, with a clear separation:

| Role | May do |
|---|---|
| **Passenger** | Request, read and cancel **their own** trip. Nothing else. |
| **Operator** | Mission control: watch, stop, rearm, reset simulation. |
| **Admin** | Accounts, roles, stored records, configuration. |

An unauthenticated request and a wrong-role request must both **fail**. A passenger's credentials
must never grant operator power, and operator credentials must not be handed to passengers.

**Select a maintained password-hashing and session library before implementing accounts.** Do not
roll your own password hashing, and do not store plaintext or unsalted digests. The existing
bearer token is a **development operator mechanism**, not passenger account management.

## Trip ownership

Adding an owner identity to trips requires a **documented schema migration** (`taxi/storage.py`),
and ownership must be validated on **read and cancel**, not just on write. Cross-passenger read and
cancel attempts must fail safely — they should not confirm that the trip exists.

Existing trip history must remain usable after the migration.

## Records and reporting

- Reports are **calculated from stored records**. No invented counts, no hard-coded business
  metrics, no figures that look measured but are not.
- Define pagination, migrations, backups and restore checks, and record them.
- A restore that has never been tested is not a backup. Test it.

## Schema discipline

`taxi/storage.py` holds the SQLite schema and the `Store` class. Changes are coordinated with
Member 2 (who coordinates shared API wiring) and with Member 1 (passenger identity). Coordinate
before changing the schema; four simultaneous schema edits is how the history becomes unreadable.

Schema migrations must be additive and idempotent where possible, and must not destroy existing
trip or event history.

---

# Part 2 — Diagnostics: honest telemetry

## The source label is not optional

Every telemetry value displayed to a user must carry:

| Field | Meaning |
|---|---|
| `source` | `"simulated"` or the device identity. Never leave it blank or implied. |
| `age_s` | How old the sample is. `age=0` alone does **not** mean healthy. |
| `valid` | Whether the sample may be used. |
| `reason` | Why it is invalid, when it is. |

The single most important rule in this file:

> **Missing or stale sensors are invalid, never substituted with fabricated healthy values.**

An absent sensor is shown as **unavailable**. It is never rendered as zero, as a default, or as a
plausible-looking number.

`taxi/comms/health.py::assess()` is to implement the shared rules. It marks a sample invalid when it
is stale, when a required key is missing, or when the source reports `sensors_ready=false`. An
invalid sample carries a **reason** and **empty** values.

## The two sources

| Source | Module | Behaviour |
|---|---|---|
| Simulation | `taxi/comms/simulated.py` (to be created) | Wraps the existing `Supervisor`. Reports `source="simulated"`. |
| Device | `taxi/comms/esp32.py` (to be created) | **Read-only.** Parses `/telemetry` and `/capture`. Cannot issue motion commands. |

The device adapter is read-only **by design**, not by omission. It parses, with a bounded parser
(`MAX_TELEMETRY_BYTES = 2048`, `MAX_TELEMETRY_FIELDS = 32`) and a hard timeout. Malformed input
yields `valid=False` with a reason — never a fabricated healthy value.

`SimulatedTelemetrySource` reports `source="simulated"` specifically so that the diagnostics
screen can distinguish it from a live device. This distinction must survive into the UI: a
simulated value shown next to a measured value, with identical styling, is misleading.

## Protocol parsing

The read-only adapters are to parse the framed protocol. `taxi/protocol/` is to contain **no I/O** — no clock,
no socket, no thread; the clock is always injected. That is what makes the parsers testable and
reusable by the firmware bridge.

When you add a parser:

- Bound the input. An unbounded parser is a denial-of-service vector on a device link.
- Bound the field count.
- Handle the timeout case as invalid, not as absent-and-therefore-fine.
- Test malformed, missing and stale samples explicitly.

## The bench firmware's honesty

`firmware/arduino/src/main.cpp` reports `motion_enabled=false` and `sensors_ready=false`. It
accepts `PING` and `STOP` and rejects everything else. **STOP receipt is not measured braking** —
the firmware has no actuator and cannot brake. Say "stop acknowledged", not "vehicle stopped".

`firmware/esp32_cam` exposes `/capture` (one JPEG), `/telemetry` (plain text with the last
unvalidated UART line and its age) and `/stop` (requests STOP on UART; 503 if UART is disabled).
Before the first UART line arrives it reports `NO_TELEMETRY`. The `/stop` endpoint acknowledges
sending a serial request — it does not confirm physical stopping.

## Hardware confirmation

No hardware is connected. Blocked on exact facts: camera module and PSRAM variant, camera pin map,
Arduino model, driver part number, steering type, encoder feedback, sensor models and pins, UART
pins with level shifting, and independent emergency-stop wiring. Record them in
`docs/HARDWARE.md`.

**Never enable physical outputs from guessed pins or simulated thresholds.** The simulator's
constants are not physical limits.

## Planned file ownership

| Path | Owner |
|---|---|
| `taxi/static/admin.html` / `admin.js` | Member 3 |
| `taxi/routers/admin.py` | Member 3 |
| `taxi/auth.py` | Member 3 |
| `taxi/static/diagnostics.html` / `diagnostics.js` | Member 4 |
| `taxi/routers/diagnostics.py` | Member 4 |
| `taxi/comms/` adapters and parsers | Member 4 |
| `taxi/storage.py` schema | Member 3, coordinated with Member 2 |

## Verification

```bash
./.venv/Scripts/python.exe -m pytest -q
./.venv/Scripts/ruff.exe check .
node --check taxi/static/admin.js        # if you create it
node --check taxi/static/diagnostics.js  # if you create it
```

Access-control tests must be present and must demonstrate that the wrong role fails.

## Reference

- `references/diagnostics.md` — telemetry sample shape, health rules, and the display contract.
- `references/ownership.md` — the file-ownership map for all four workstreams.
- `docs/TEAM-WORKSPACE.md` — Members 3 and 4's task lists.
- `docs/PROTOCOL-v2.md` — the binary protocol the adapters parse.
