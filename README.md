# Campus Taxi — Python + ESP32-CAM + Arduino

A runnable starting repository for the autonomous campus taxi project. The laptop application runs an A-to-B simulation, FastAPI backend, operator dashboard, telemetry and SQLite history. No ROS is required.

**v0.1 is a simulation and bench-communications foundation. Physical navigation, sensor drivers and motor actuation are not implemented.** The board starters never enable motors and the laptop rejects `TAXI_MODE=hardware`. Nothing in this release is approved for carrying passengers or road use.

## Run on Windows

Install Python 3.10+ and Git, extract this folder, then open PowerShell inside it:

```powershell
py -3.10 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -e ".[dev]"
Copy-Item .env.example .env
.\.venv\Scripts\python.exe -m taxi
```

Open **http://127.0.0.1:8000**. The operator token is `local-dev-only` unless changed in `.env`. Python 3.11 or 3.12 also works: select the installed version when creating the environment. Activation is unnecessary. API reference: http://127.0.0.1:8000/docs.

Linux / macOS:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -e '.[dev]'
cp .env.example .env
.venv/bin/python -m taxi
```

Run one server process, one worker, and one vehicle supervisor. Do not add multiple Uvicorn workers: each would create a separate simulated vehicle. The default command listens only on loopback. Internet access is not needed after installing dependencies.

## Try the full demo

1. Start at **A — Engineering**, select **B — Library**, and start the simulated trip.
2. Watch the vehicle follow A → J → B. The 10 m virtual journey takes roughly 13 seconds.
3. Inject an obstacle. The mission becomes FAULTED and the vehicle stops.
4. Clear the obstacle. The fault stays latched until you press Rearm.
5. A vehicle stopped between named stops cannot start another mission. Reset to A for the next test; this teleports only the simulator.
6. Inject frozen sensors or a lost link. Updates expire after 0.5 seconds and fault the trip.
7. Try A → C, inspect history, then stop/restart the server. Active trips become INTERRUPTED; a server restart never resumes a mission.

All positions, distances and fault inputs in the dashboard are simulated. Speed / timing constants are demonstration values, not physical vehicle settings. The Stop button is a software simulation stop, not a physical emergency stop.

## What is included

| Component | Implemented now | Next integration |
|---|---|---|
| Dashboard | Route map, trips, status, fault injection, stop/rearm/reset, history | Passenger accounts and real vehicle telemetry |
| Backend | Validated API, operator token, read-only telemetry WebSocket, SQLite | User roles, deployment authentication, dispatch |
| Brain | Route graph, deterministic simulated motion, stale-input gates and latched faults | Real perception, localization, closed-loop control |
| Protocol | Newline-ASCII bench protocol only, on the ESP32-CAM relay | Binary COBS + CRC-8 framing and versioned motion frames (specified in [docs/PROTOCOL-v2.md](docs/PROTOCOL-v2.md), not yet implemented) |
| Perception | None — the dashboard renders the route graph, not camera frames | Frame source, detector backend, costmap and VFH scoring |
| Comms | Supervisor-backed simulated telemetry over the existing WebSocket | Read-only device telemetry adapter with bounded parsing |
| ESP32-CAM | AI Thinker camera capture + read-only UART telemetry + stop-request firmware starter. **Compiles.** | Legacy bench fallback; confirm board, pins, credentials |
| Arduino | Uno bench parser only — newline-ASCII line handling, no sensors, no motors. **Compiles.** | Sonar FSM, subsumption arbiter, watchdog, kinematics — no actuator pins until the board and driver are confirmed |
| ESP32-S3 | Not present in this repository | Primary edge target: FreeRTOS tasks, camera DMA, WS client, UART bridge |
| AI models | Optional vision dependency group; no weights bundled | Benchmark/select a commercially suitable checkpoint |

## The three-tier target architecture

The next version moves control across three tiers:

| Tier | Hardware | Owns |
|---|---|---|
| 1. Perception & Planning | Laptop (Python) | camera → detections → costmap → VFH → `(v, ω)` |
| 2. Edge Gateway | ESP32-S3 (FreeRTOS) | camera DMA, WebSocket client, telemetry forwarding, UART bridge |
| 3. Actuation & Safety | Arduino Uno R3 | sonar FSM, subsumption arbiter, watchdog, motor PWM |

**Safety authority flows downward only.** The laptop may *request* motion; only the Arduino may
*grant* it. Losing the laptop or the Wi-Fi must always resolve to stop.

This is specified in **[docs/ARCHITECTURE-v2.md](docs/ARCHITECTURE-v2.md)** and
**[docs/PROTOCOL-v2.md](docs/PROTOCOL-v2.md)**, with the decisions recorded in
**[docs/adr/](docs/adr/)**. **None of these tiers is implemented yet** — the documents are a
specification to build against, not a description of existing code. The current tree contains
no protocol, perception, planning or mission modules, and no ESP32-S3 target.

## Project layout

```text
taxi/
  api.py                 REST, WebSocket, application lifecycle
  supervisor.py          Mission state machine and simulator
  navigation.py          Stops and shortest permitted routes
  storage.py             SQLite trips and events
  static/                Browser dashboard (no Node build needed)
firmware/
  esp32_cam/             Bench starter: AI Thinker camera / UART relay
  arduino/               Uno bench starter: newline-ASCII line parser
scripts/
  setup.ps1              Optional Windows installer
  camera_probe.py        Authenticated JPEG capture test
tests/                   Supervisor and API tests
docs/                    Architecture, protocol, ADRs, hardware, validation, roadmap
.agents/skills/          Six project skills — core, hardware, perception, passenger,
                         operator, admin/diagnostics
```

## Project skills

`.agents/skills/` holds six project-scoped skills, one per workstream plus the shared core. Each
carries the invariants for its area and the file-ownership map, so parallel work does not
collide:

| Skill | Covers |
|---|---|
| `taxi-core` | Mission state, latched faults, terminal statuses, the non-negotiable invariants |
| `taxi-hardware` | ESP32-S3 / Arduino bench work, framing contract, hardware confirmation checklist |
| `taxi-perception` | Frames, detectors, costmap, freshness, the "absence of evidence" rule |
| `taxi-passenger` | Booking, live progress, arrival promise, cancellation, ownership |
| `taxi-operator` | Supervision, stop/rearm/reset semantics, shared contracts, design tokens |
| `taxi-admin-diagnostics` | Accounts, roles, migrations, honest telemetry display |

## Checks

```powershell
mkdir -p runtime/pytest-tmp
.\.venv\Scripts\python.exe -m pytest -q --basetemp=runtime/pytest-tmp
.\.venv\Scripts\ruff.exe check .
node --check taxi/static/app.js
```

`--basetemp` is required on machines where pytest's default temp directory is not writable;
without it the `tmp_path` test in `tests/test_supervisor.py` errors with `PermissionError`. See
[docs/VALIDATION.md](docs/VALIDATION.md) for the verified results of each check.

## Clone from GitHub

```bash
git clone https://github.com/praisecyber/Autonomous-Taxi.git
cd Autonomous-Taxi
```

Follow the Windows or Linux setup above, then continue work with Codex. Commit and push your changes from this folder. `.env`, firmware `secrets.h`, recordings, model weights, runtime data and virtual environments are excluded from Git. Do not commit credentials.

## Continue with Codex

Read `AGENTS.md` and `docs/ROADMAP.md`. First run the tests, then tackle one milestone at a time. Hardware motion stays unavailable until the exact board, driver, steering method, feedback sensors and emergency-stop wiring are confirmed. Do not replace missing sensors with invented healthy readings.

This repository does not declare a redistribution licence for your original application code; choose one before publishing it for third-party reuse. Dependencies and any future model weights retain their own licences. No pretrained AI weights are included.
