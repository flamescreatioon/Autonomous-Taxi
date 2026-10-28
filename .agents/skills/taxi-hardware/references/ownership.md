# File ownership map


> **Status (2026-10-28).** The invariants below are the rules the implementation must satisfy.
> The **enforcement tests named here do not exist yet** — `tests/test_safety_invariants.py`,
> `tests/test_protocol_freshness.py` and the other files referenced are to be written along with
> the v2 modules. The rules themselves are binding regardless of when the tests arrive.

Adapted from `docs/TEAM-WORKSPACE.md`. The point of this map is to stop four people editing the
same file at the same time and discovering it at merge time.

---

## Shared — coordinate before editing

| File | Coordinator | Rule |
|---|---|---|
| `taxi/api.py` | Member 2 | Route registration only. Prefer a small router module per workstream over growing this file. |
| `taxi/storage.py` | Member 3 | Schema changes coordinated with Member 2. |
| `taxi/static/shared.css` | Member 2 | Shared design tokens. Does not exist yet — create once, properly. |
| `taxi/static/api-client.js` | Member 2 | Shared fetch/WebSocket client. Does not exist yet. |
| `taxi/static/index.html`, `taxi/static/app.js` | Member 2 | The existing dashboard. Splitting it into `/operator` must not break the A-to-B simulation. |

---

## Member 1 — Passenger experience

Branch `feature/passenger-interface` · Interface `/passenger`

| Path | Status |
|---|---|
| `taxi/static/passenger.html` | Planned |
| `taxi/static/passenger.js` | Planned |
| `taxi/routers/passenger.py` | Planned |
| `tests/test_passenger.py` | Planned |

Reuse: `taxi/static/index.html`, `taxi/static/app.js`, `GET /api/stops`, `POST /api/trips`,
`GET /api/trips/{id}`, `POST /api/trips/{id}/cancel`.

---

## Member 2 — Operator and mission management

Branch `feature/operator-interface` · Interface `/operator`

| Path | Status |
|---|---|
| `taxi/static/operator.html` | Planned |
| `taxi/static/operator.js` | Planned |
| `taxi/routers/operator.py` | Planned |
| `taxi/api.py` route registration | Shared — coordinated |

Reuse: `taxi/supervisor.py`, `taxi/navigation.py`, `taxi/api.py`, the operator stop/rearm endpoints,
simulation reset/hazards, the existing dashboard.

---

## Member 3 — Administration, accounts and records

Branch `feature/admin-interface` · Interface `/admin`

| Path | Status |
|---|---|
| `taxi/static/admin.html` | Planned |
| `taxi/static/admin.js` | Planned |
| `taxi/routers/admin.py` | Planned |
| `taxi/auth.py` | Planned |
| `tests/test_auth.py`, `tests/test_admin.py` | Planned |
| database migration scripts | Planned |

Reuse: `taxi/storage.py`, SQLite trips/events, `GET /api/trips`, `GET /api/events`, the
development operator-token protection.

---

## Member 4 — Vehicle diagnostics, firmware and communication

Branch `feature/diagnostics-interface` · Interface `/diagnostics`

| Path | Status |
|---|---|
| `taxi/static/diagnostics.html` | Planned |
| `taxi/static/diagnostics.js` | Planned |
| `taxi/routers/diagnostics.py` | Planned |
| `taxi/comms/` | **Not yet implemented** — to be created as `base.py`, `health.py`, `simulated.py`, `esp32.py` |
| `tests/test_comms.py` | **Not yet implemented** |

Reuse: `firmware/esp32_cam/`, `firmware/arduino/`, `firmware/esp32_s3/`,
`scripts/camera_probe.py`, `docs/HARDWARE.md`, `docs/PROTOCOL.md`, `docs/PROTOCOL-v2.md`.

---

## Shared engineering ownership — all members

| Path | Owner |
|---|---|
| `taxi/protocol/` | Shared contract — changes require agreement between all four |
| `taxi/mission/` | Shared contract |
| `taxi/perception/`, `taxi/planning/` | Member 2 with Member 4 |
| `firmware/` | Member 4; changes to the protocol affect all members |
| `docs/adr/` | Agreement required to supersede an ADR |
| `tests/test_safety_invariants.py` | Do not weaken. It is the guardrail for everyone. |

---

## Pull request rules

- Preserve the working simulator. Use a separate feature branch for each workstream.
- Avoid simultaneous edits to shared `api.py` / `storage.py` / styles. Coordinate module
  boundaries first.
- Run pytest and Ruff, run JS syntax checks, and record browser checks for changed screens.
- Hardware claims require compiler **and** bench logs. Simulation alone is not physical validation.
- Do not commit `.env`, Wi-Fi/device credentials, recordings or model weights.
