# Suggested build roadmap

Consolidated delivery plan from the current v0.1 simulation to a supervised, low-speed physical campus taxi. It merges the previous engineering milestone plan with the four-person task board in [TEAM-WORKSPACE.md](TEAM-WORKSPACE.md); that file stays the per-member checklist with owners and evidence links. Numbering here follows the team board (M0–M7), which supersedes the older M0–M5 numbering previously used in this file.

Work runs on two tracks that merge at M6:

- **Track A — Software (laptop + UI):** M1 → M2 → (parts of M5 in parallel) → M6. Starts immediately, needs no hardware.
- **Track B — Hardware (boards + vehicle):** M3 → M4 → M6. Blocked until actual boards and wiring are confirmed in [HARDWARE.md](HARDWARE.md).

Dates are unassigned. Hardware milestones are gated by facts and measured tests, not by calendar or wish.

## Where the project stands — verified facts

Established (see [VALIDATION.md](VALIDATION.md)):
- Python/FastAPI backend, operator dashboard, A→B/C shortest-route simulation, SQLite trips/events.
- Latched faults, explicit rearm, terminal trip statuses, one-supervisor lifecycle.
- 14 pytest cases pass locally on Python 3.12; Ruff clean; JS syntax check clean.
- GitHub Actions configured for 3.10/3.12; a passing run is linked in TEAM-WORKSPACE.md.

Not established (do not claim otherwise):
- Neither firmware target has compiled successfully in the development environment.
- No browser/visual QA beyond API and static checks.
- No physical hardware connected: no camera test, no UART link, no sensors, no motor control, no measured stopping.

## Hard rules that gate every milestone

- Preserve the runnable simulation; every software milestone keeps it working.
- One server process, one supervisor. Reconnect never authorizes motion.
- Keep terminal trip statuses and latched faults; clearing a hazard never resumes a trip.
- Application stop and physical emergency stop stay separate concepts.
- Never enable physical outputs from guessed pins, driver types or thresholds. Record confirmed hardware in HARDWARE.md first.
- Missing or stale sensors are invalid — never substituted with fabricated healthy values.
- Compilation is not a hardware test; a simulation pass is not physical validation. Report exactly which were run.

## Milestone overview

| Milestone | Track | Depends on | Exit gate |
|---|---|---|---|
| M0 Foundation | — | — | Done: tests + live simulated A→B check, CI green. |
| M1 Shared shell and contracts | A | M0 | Four pages open, share styling, handle errors; simulator still runs. |
| M2 Connected four-interface simulation | A | M1 | One authorized passenger trip visible consistently on all four screens; access tests pass. |
| M3 Confirmed hardware, read-only telemetry | B | hardware facts | Real values carry source/age/validity; missing or stale never looks healthy. |
| M4 Independent local control and stop | B | M3 | Bench fault tests and measured stopping/disable results recorded before any autonomy. |
| M5 Perception and localization | A+B (experiments may start during M1/M2) | recorded data; M4 before control integration | Required-object detection, localization and latency meet measured project limits. |
| M6 Supervised physical A-to-B trial | merge | M2 + M4 + M5 | Approved course: reach the stopping area, obey route/speed bounds, pass fault tests. |
| M7 Service and fleet expansion | A | M6 (features may be prototyped earlier on simulation) | Each feature has approved scope, hardware and recorded acceptance evidence. |

## M0 — Delivered foundation (complete)

Python package, validated API with operator token, dashboard, SQLite history, read-only telemetry WebSocket; A→B/C simulation with latched faults and explicit recovery; supervisor/API tests; ESP32-CAM and Arduino bench firmware starters (never armed).

## M1 — Shared shell and contracts (next; start immediately, no hardware needed)

**Goal:** four interfaces exist as real pages sharing one design system and one API contract.

- Create `shared.css` design tokens (colors from TEAM-WORKSPACE), `api-client.js`, shared navigation.
- Stand up `/passenger`, `/operator`, `/diagnostics`, `/admin` pages with per-workstream routers; split the existing dashboard into the operator page without breaking the demo.
- Document shared contracts: trip statuses, role permissions, error shapes, telemetry source labels (simulated vs device), freshness fields.
- Keep one supervisor; avoid four people editing `api.py`/`storage.py` in parallel.

**Exit gate:** all four pages open, share styling, handle loading/error/disconnected states; original simulation demo still passes.

**Suggested Codex prompt:**
> Read AGENTS.md and docs/TEAM-WORKSPACE.md. Create the shared UI shell (design tokens, api-client, navigation) and the four page/router skeletons with stub content. Keep the existing dashboard demo runnable and add a smoke test that all four pages and their routes respond.

## M2 — Connected four-interface simulation

**Goal:** one simulated mission flows through all four screens with real accounts and ownership.

- Member 3: accounts/roles (passenger, operator, admin) with a maintained hashing/session library; schema migration adding trip owner; ownership enforced on read/cancel; role/ownership tests.
- Member 1: booking, live progress from read-only telemetry, cancellation — no shared operator token published to passengers.
- Member 2: operator supervision of passenger-initiated missions; preserve one-active-mission, terminal cancel, fault latch, rearm.
- Member 4: diagnostics page with honest source labels — simulation vs device, sensor age, validity, stop reasons.

**Exit gate:** one authorized passenger trip appears consistently across passenger/operator/diagnostics/admin; wrong-role and cross-passenger access tests fail safely.

**Suggested Codex prompt:**
> Read AGENTS.md and docs. Implement passenger accounts, roles and trip ownership with a schema migration and access-control tests. Existing operator token flow and all current supervisor tests must keep passing.

## M3 — Confirmed hardware and read-only telemetry

**Goal:** real boards on the bench feeding honest telemetry into the laptop. Physical outputs stay disabled.

- Record actual boards, pins, driver, steering, feedback sensors, UART wiring (level shifting, boot pins) and independent stop wiring in HARDWARE.md.
- Compile both firmware targets in PlatformIO; record compiler logs (first successful build is itself a milestone — none exists yet).
- Bench test: one JPEG capture via `camera_probe.py`, PING/STOP relay, UART telemetry line with age.
- Implement real sensor drivers with timestamps and validity; Python read-only adapter with a bounded parser.
- Tests: malformed frames, missing sensors, stale samples, timeout — none may render as healthy.

**Exit gate:** diagnostics shows measured values with source/age/validity; missing or stale data can never look healthy.

**Suggested Codex prompt:**
> Read AGENTS.md and docs. Implement a read-only adapter for the confirmed ESP32 telemetry format with malformed/stale-data tests. Preserve the simulation and do not enable motors.

## M4 — Independent local control and stop

**Goal:** the vehicle can be commanded and stopped on the bench, with safety owned locally by the Arduino — before any autonomous navigation.

- Implement the confirmed motor driver and a hardware-disable interface on the bench.
- Build the future motion protocol from [PROTOCOL.md](PROTOCOL.md): versioned frames with session ID and sequence, end-to-end freshness (no queued command gaining new lifetime), Arduino-owned watchdog, latched fault/rearm handshake.
- Fault-injection tests for the complete host → ESP32 → Arduino path: duplicates, old sessions, reorder, partial/oversized frames, stale sensors, board reset, Wi-Fi loss, stalled laptop.
- Measure actual stopping (STOP vs coast/brake) and record it. Choose all physical thresholds from these measurements, never from simulator constants.

**Exit gate:** bench fault tests and measured stopping/disable results recorded before any autonomous motion is attempted.

## M5 — Perception and localization

**Goal:** the laptop knows where the vehicle is and what is ahead, within measured limits. Data collection and offline experiments may start during M1/M2; control integration waits for M4.

- Collect and label camera recordings with appropriate permissions.
- Detect route markers and path boundaries (OpenCV tests first); benchmark YOLOX Tiny vs RT-DETRv2-S with exact licence-checked weights — none are bundled today.
- Add encoders or another tested localization method for progress between markers.
- Monocular relative depth is not stopping-distance ground truth.
- Serve perception from bounded latest-frame queues outside the supervisor loop; preserve freshness through every hop.

**Exit gate:** required-object detection, localization accuracy and latency meet measured project limits on the recorded dataset.

## M6 — Supervised physical A-to-B trial

**Goal:** first real mission on an approved low-speed course, integrating the proven stack.

- Connect tested perception, localization and controller to the M4-proven communication/safety layer.
- Start with stop-and-review obstacle handling; operator supervises every run.
- Test loss of camera, localization, serial, Wi-Fi and laptop process during motion.
- Report measured response times and stopping distances for declared conditions.

**Exit gate:** vehicle reaches the defined stopping area, obeys route/speed bounds, and passes sensor/link/obstacle fault tests. Evidence: trigger, expected behavior, actual result, logs, remaining gaps.

## M7 — Service and fleet expansion

Pickup dispatch, boarding sensors, battery measurement, maintenance records, accessibility, notifications, multi-vehicle, optional payments — each with its own approved scope and acceptance evidence. Passenger carrying and any public-road operation need separate system validation and applicable approvals.

## Suggested sequencing

1. **Now:** M1 shared shell (all members, coordinated by Member 2) while Member 4 starts M3 hardware confirmation in parallel — buying/identifying boards blocks the whole B track.
2. **Next:** M2 connected simulation on Track A; M3 compile + bench telemetry on Track B as soon as hardware facts land.
3. **Parallel anytime:** M5 data collection, labeling and offline detector experiments (no vehicle needed).
4. **Gated:** M4 only after M3; M6 only after M2 + M4 + M5; M7 features prototyped on simulation are fine, physical features are not.

## Completion evidence rules

- Per milestone, document trigger, expected behavior, actual result, logs and remaining gaps.
- Link evidence in TEAM-WORKSPACE.md and tick the task there, in a reviewed PR.
- Firmware claims need compiler and bench logs; driving claims need physical measurements. Compilation is not a hardware test; a simulation pass is not physical validation.
