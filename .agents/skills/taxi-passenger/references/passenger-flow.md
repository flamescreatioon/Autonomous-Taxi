# Passenger flow


> **Status (2026-10-28).** The invariants below are the rules the implementation must satisfy.
> The **enforcement tests named here do not exist yet** — `tests/test_safety_invariants.py`,
> `tests/test_protocol_freshness.py` and the other files referenced are to be written along with
> the v2 modules. The rules themselves are binding regardless of when the tests arrive.

---

## The states a passenger sees

| Server status | What the passenger sees |
|---|---|
| No trip | The booking screen: current stop, destination selector, request button. |
| Request sent, not yet confirmed | "Requesting…" — the button disabled, a spinner. |
| `ACTIVE` | The trip card: origin, destination, live progress. |
| `COMPLETED` | "Your taxi has arrived" — **only** if speed is zero. |
| `CANCELLED` | "Your trip was cancelled." Terminal. |
| `FAULTED` | "The trip was interrupted by a fault." Terminal, and honest. |
| `INTERRUPTED` | "The trip was interrupted." Terminal. |

Distinguish every one of these in the user's own words. Do not surface the raw enum names.

---

## Booking

1. Show the vehicle's **current stop**.
2. Fetch the stop list from `GET /api/stops`.
3. The passenger selects a **destination only**. The origin is the vehicle's current stop, not a
   passenger input.
4. `POST /api/trips` with the destination.
5. Handle the failure cases explicitly:
   - same stop — rejected,
   - invalid stop — rejected,
   - vehicle busy — rejected,
   - hazards or stale telemetry — rejected.

The error message must say which one. "Something went wrong" is useless to someone standing on a
campus road.

**A double-click must not silently create a second active mission.** There is one vehicle and one
active trip. Disable the button on submit, and make the request idempotent from the client's
point of view.

---

## Live progress

Consume the read-only telemetry WebSocket and match it to the **active trip id**.

```javascript
if (snapshot.trip && snapshot.trip.id === myTripId) {
  // this snapshot is about my trip
}
```

If the ids do not match, show nothing. Showing another trip's progress is worse than showing no
progress.

**Disconnection is a state, not a silence.** A WebSocket that stops updating without saying so
leaves the passenger looking at a frozen progress bar and assuming the taxi has stalled. Show a
"connection lost — reconnecting" state, and re-fetch the trip status on reconnect rather than
trusting the last known snapshot.

### The arrival promise

```javascript
const arrived = trip.status === "COMPLETED" && snapshot.speed === 0;
```

Both conditions. A completed trip at non-zero speed is a bug in the backend, and the UI must not
render arrival in that state. If you find such a case, report it — do not paper over it.

---

## Cancellation

Two distinct moments:

1. **Request sent** — the server accepted the request. The vehicle still has to respond.
2. **Cancelled** — the server confirmed the trip reached the terminal `CANCELLED` status.

Render "Requesting cancellation…" between them, not "Cancelled". And never imply the vehicle has
stopped — the application stopping is not the vehicle stopping.

A cancelled trip stays cancelled. It does not return to `ACTIVE`.

---

## Identity

Passengers authenticate as passengers, with their own session. **The shared operator token is never
given to a passenger.** It is a development mechanism for a single operator console.

The server enforces ownership on read and cancel. The UI hides the controls it should not show,
but that is an affordance, not a control — the server check is the control.

A wrong-role or wrong-owner request must fail without confirming that the trip exists.

---

## Layout requirements

- **Phone width first.** Outdoors, one-handed, possibly in a hurry. Test at 360 px before
  1440 px.
- **Large tap targets** for the destination selector and the request button.
- **Loading, error and disconnected** states on every screen that fetches.
- **Do not rely on colour alone.** Pair every status colour with a word.

Shared design tokens live in `taxi/static/shared.css` (planned). Use them rather than introducing
new colours.

---

## API contract

| Method | Path | Use |
|---|---|---|
| `GET` | `/api/stops` | The configured stops |
| `POST` | `/api/trips` | Request a trip — send the **destination only** |
| `GET` | `/api/trips/{id}` | Poll a trip's status |
| `POST` | `/api/trips/{id}/cancel` | Request cancellation |
| `WS` | the read-only telemetry socket | Live progress |

Full endpoint definitions are in `taxi/api.py` and the OpenAPI docs at `/docs`.
