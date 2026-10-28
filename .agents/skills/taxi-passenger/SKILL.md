---
name: taxi-passenger
description: Use this skill for any work on the Autonomous Taxi's passenger experience - booking, ride request, trip request, pickup, arrival, destination selection, cancellation, passenger UI, the /passenger page, passenger.js, passenger.html, taxi/routers/passenger.py, trip ownership, passenger authentication, passenger sessions, or the visible states of a passenger's own trip. Trigger when someone asks how a passenger books a ride, sees trip progress, cancels a trip, is told the taxi has arrived, or when touching code that creates or reads a trip on behalf of a person rather than an operator.
---

# Taxi Passenger — booking, progress and the promise the UI makes

> **Repository state (2026-10-28).** The three-tier architecture these skills describe is
> **specified but not implemented**. There is no `taxi/protocol/`, `taxi/comms/`,
> `taxi/perception/`, `taxi/planning/` or `taxi/mission/` package, no `firmware/esp32_s3/`
> target, and no binary framing code on either board. What exists is the v0.1 simulation, the
> ESP32-CAM bench starter and the Arduino newline-ASCII bench parser. Treat every path and
> constant named below as **the target design**, not as code you can read or run. See
> `docs/VALIDATION.md` for the verified state of the current tree.


The passenger interface has one job: tell the truth about where the taxi is and what has happened
to their trip. Everything else follows from that.

## The arrival promise

> **Arrival appears only when the correct trip is `COMPLETED` **and** speed is zero.**

Neither condition alone is sufficient. A trip marked completed while the vehicle is still moving
is a bug, and the UI must not render "Your taxi has arrived" in that state.

Match the live telemetry to the **active trip id**, not to "the current trip" or "the most recent
trip". If the ids do not match, show nothing rather than showing the wrong trip's progress.

## A cancellation request is not a confirmed stop

When a passenger taps cancel, the server accepts a *request*. The vehicle still has to respond.
The UI must distinguish:

- **Request sent** — the request was accepted by the server.
- **Cancelled** — the server confirmed the trip reached the terminal `CANCELLED` status.

Never render "Your trip has been cancelled" at the moment the request is sent, and never imply
the vehicle has stopped. Terminal statuses are final — once cancelled, the trip stays cancelled.

## Identity and ownership

Passengers authenticate as passengers. They must **never** be given the shared operator token —
that token is a development mechanism for a single operator console, not passenger account
management.

A passenger may read and cancel **their own** trip and nothing else. Cross-passenger access must
fail safely, not fail informatively — a wrong-role request should not confirm that a trip exists.

Ownership is enforced in the server, not by hiding buttons. Hiding a button is a UI affordance;
the server check is the actual control.

## UI requirements

- **Phone-width first.** The passenger will be holding the phone, outdoors, possibly one-handed.
  Test at a narrow viewport before a wide one.
- **Explicit states.** Every screen needs distinct **loading**, **error** and **disconnected**
  handling. A WebSocket that silently stops updating is worse than one that says it disconnected —
  the passenger needs to know the progress they are reading is current.
- **Distinguish every trip status** in the user's own words: accepted, active, stopped, faulted,
  completed. Do not surface the raw enum names without explanation.
- **Error messages must be useful.** "No taxi available" is not the same as "A taxi is already
  serving another passenger", and the passenger can act on the second one.

## Existing endpoints to reuse

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/stops` | The configured stop list. Send the **destination only** on a request. |
| `POST` | `/api/trips` | Request a trip. |
| `GET` | `/api/trips/{id}` | Poll a trip's status. |
| `POST` | `/api/trips/{id}/cancel` | Request cancellation. |

A double-click must not silently create a second active mission. There is one vehicle.

## One vehicle, one active trip

The vehicle is a shared resource with a single active mission. A request for an unavailable
vehicle must return a clear, honest error rather than queueing silently. There is one vehicle.

## Planned file ownership

These paths are **planned and do not exist yet**. Create them; do not modify a shared file to host
passenger code.

| Path | Owner |
|---|---|
| `taxi/static/passenger.html` | Member 1 |
| `taxi/static/passenger.js` | Member 1 |
| `taxi/routers/passenger.py` | Member 1 |
| `tests/test_passenger.py` | Member 1 |

Shared files (`taxi/static/shared.css`, `taxi/static/api-client.js`, `taxi/api.py` route
registration) are coordinated by Member 2. Coordinate before touching them.

## Verification

```bash
./.venv/Scripts/python.exe -m pytest -q
./.venv/Scripts/ruff.exe check .
node --check taxi/static/passenger.js   # if you create it
```

Record a browser check at both a phone width and a desktop width. See
`docs/TEAM-WORKSPACE.md` for the full task list and exit gates.

## Reference

- `references/passenger-flow.md` — the passenger flow, states and API contract.
- `references/ownership.md` — the file-ownership map for all four workstreams.
- `docs/TEAM-WORKSPACE.md` — Member 1's task list.
