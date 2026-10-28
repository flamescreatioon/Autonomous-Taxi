# Codex project instructions

## Goal and current scope
Build a campus taxi using Python on a laptop, an ESP32-S3 edge gateway for camera / communication and an Arduino for local sensor / motor control. No ROS dependency. Read README.md and docs/ROADMAP.md before substantial work.

**v0.1 is the only implemented version.** It provides a simulation, backend, dashboard and two bench-only firmware starters (ESP32-CAM, Arduino Uno). The three-tier architecture — a binary protocol, a read-only telemetry adapter, a perception/planning seam and disarmed firmware tiers — is **specified but not implemented**: there is no `taxi/protocol/`, `taxi/comms/`, `taxi/perception/`, `taxi/planning/` or `taxi/mission/` package, and no `firmware/esp32_s3/` target. Physical navigation and actuation are NOT implemented. Do not represent simulated data as real telemetry.

The target architecture is docs/ARCHITECTURE-v2.md, with decisions in docs/adr/. The ESP32-CAM remains a supported bench fallback; the ESP32-S3 is the primary target.

## Work rules
- Run `python -m pytest -q` and `ruff check .` after Python changes. In this environment use the project venv: `./.venv/Scripts/python.exe -m pytest -q --basetemp=runtime/pytest-tmp` and `./.venv/Scripts/ruff.exe check .` — `ruff` and `python` are not on PATH, and pytest's default temp directory is permission-broken here, so `--basetemp` is required for the `tmp_path` test in `tests/test_supervisor.py`.
- Run `node --check taxi/static/app.js` after browser JS changes.
- Keep dependencies in pyproject.toml. Keep package discovery limited to taxi.
- Keep secrets and generated runtime data out of Git.
- One server process owns one supervisor. Do not add multiple API workers.
- Preserve terminal trip statuses and latched faults. Reconnect never authorizes motion.
- Maintain separate application stop and physical emergency-stop concepts.
- Never enable physical outputs with guessed pins, driver types or safety thresholds.
- Before motor integration, record confirmed hardware in docs/HARDWARE.md and add fault-injection tests for the complete host → ESP32 → Arduino protocol.
- Missing / stale sensors are invalid, never substituted with fabricated healthy values.
- Firmware target builds and hardware tests are separate: report exactly which were run.
- Simulator constants must never become physical thresholds. The 300 ms watchdog and the 20/15 cm emergency distances are provisional until measured (docs/adr/ADR-0006.md).
- Protocol code must contain no I/O — no clock, socket or thread. The clock is always injected, so the same implementation is testable on the host and reusable from a board-side bridge. When the v2 framing is implemented, the Python and C++ implementations must be changed together and compared byte for byte by a shared-vector test; do not let them drift.

## Project skills
.agents/skills/ holds six project-scoped skills: taxi-core, taxi-hardware, taxi-perception, taxi-passenger, taxi-operator, taxi-admin-diagnostics. Read the relevant one before working in its area.

## Next milestone
Implement the read-only hardware telemetry adapter and its parser / timeout tests, after confirming the actual board and UART pin assignments in docs/HARDWARE.md. It must be read-only: it parses telemetry and never issues motion commands. Preserve the runnable simulation throughout, and keep the v2 documents (docs/ARCHITECTURE-v2.md, docs/PROTOCOL-v2.md, docs/adr/) as the specification to build against.

See docs/PROTOCOL-v2.md §7 for the full accounting of what is wired and what is not.
