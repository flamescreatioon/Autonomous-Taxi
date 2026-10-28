---
name: taxi-perception
description: Use this skill for any work on the Autonomous Taxi's perception stack - camera frames, JPEG, OpenCV, YOLO, detectors, lane detection, path boundary detection, markers, bounding boxes, costmap or occupancy grid inputs, sensor projection, depth, monocular depth, localization, encoders, position tracking, latest-frame queues, frame staleness, stale frames, inference, frame drops, or anything touching taxi/perception/ and taxi/planning/. Trigger before adding a detector backend, before wiring the camera into the pipeline, and before changing how a frame becomes an obstacle.
---

# Taxi Perception — frames, detectors, costmap and localisation

> **Repository state (2026-10-28).** The three-tier architecture these skills describe is
> **specified but not implemented**. There is no `taxi/protocol/`, `taxi/comms/`,
> `taxi/perception/`, `taxi/planning/` or `taxi/mission/` package, no `firmware/esp32_s3/`
> target, and no binary framing code on either board. What exists is the v0.1 simulation, the
> ESP32-CAM bench starter and the Arduino newline-ASCII bench parser. Treat every path and
> constant named below as **the target design**, not as code you can read or run. See
> `docs/VALIDATION.md` for the verified state of the current tree.


The perception stack has three hard properties: it must stay **fresh**, it must stay **bounded**,
and it must never present absence of evidence as evidence of absence.

## Freshness beats completeness

A late frame is worse than no frame. The pipeline therefore keeps a **single-slot** queue, not a
backlog: a frame that is superseded before inference starts is discarded and counted.

`taxi/perception/pipeline.py` is to provide:

- `LatestFrameQueue` — holds one frame, drops the previous one, counts drops.
- `PerceptionPipeline` — discards frames older than `max_age_s` **before** handing them to the
  backend.

If you add buffering, retry logic, or a multi-frame batch anywhere in this path, you have broken
the freshness guarantee. A backlog does not make perception more accurate; it makes it describe the
past.

## Absence of evidence is not evidence of absence

This is the rule most easily broken by accident:

> **Missing or stale sensors are invalid, never substituted with fabricated healthy values.**

Concretely, in this codebase:

- `SensorReading.distance_cm = None` marks **nothing** in the occupancy grid. Not free. Not
  occupied. `None` means "no evidence".
- `PerceptionPipeline` drops a stale frame rather than delivering it.
- `taxi/comms/health.py::assess()` marks a telemetry sample invalid when it is stale or when a
  required key is missing, and carries a reason with it.
- The emergency reflex fires only on a **valid** measurement below threshold. A missing sensor
  does not trigger it.

A detector that returns no detections must not be interpreted as "the path is clear". If the
system cannot see, the correct output is "I cannot see" — and the tiers above must respond to
that.

## Current state — do not overstate it

| Component | Status |
|---|---|
| `taxi/perception/base.py` — `Frame`, `Detection`, `Observation`, `PerceptionBackend` | **Not yet implemented** — to be created |
| `taxi/perception/pipeline.py` — `LatestFrameQueue`, `PerceptionPipeline` | **Not yet implemented** — to be created |
| `taxi/perception/detectors.py::NullBackend` | **Not yet implemented** — to be created, and to be the only enabled backend |
| `taxi/perception/detectors.py::OpenCVBackend` | **Not yet implemented** — to be a stub raising `NotImplementedError` |

**No detector weights are bundled in this repository and none are downloaded at import or run
time.** `*.pt` and `*.onnx` are gitignored. If a task needs real detection, the weights must be
added deliberately, with their provenance recorded — never fetched silently at runtime.

`NullBackend` returning no detections is a placeholder, not a working detector. Anything that
consumes it must handle the "no evidence" case correctly.

## Monocular depth is not stopping distance

Depth estimated from a single camera is a **planning hint**, never a safety input. The Arduino's
emergency reflex uses sonar; the perception stack must not attempt to override it, and a
confidence-scaled detector output must not be treated as a distance measurement for braking.

If the project needs distance for safety, it needs a sensor that measures distance.

## Planning inputs

| Path | Role |
|---|---|
| `taxi/planning/occupancy.py` (to be created) | `OccupancyGrid`, `SensorReading`. Out-of-grid points are treated as occupied — the planner must not reason about free space it cannot see. |
| `taxi/planning/vfh.py` (to be created) | `compute_candidate_direction`. Returns `None` when **every** candidate is blocked. |
| `taxi/planning/controller.py` (to be created) | `Controller` with acceleration clamping; a large heading error reduces linear speed. |

`compute_candidate_direction` returning `None` is a feature. The planner must be able to say "I have
no valid heading". If you change it to always return the least-bad candidate, you have removed the
only signal that the path is blocked.

## Localisation

There is no localisation in the current codebase. The v0.1 navigation uses a **virtual** route
graph (`taxi/navigation.py`) with ideal geometry — no vehicle dynamics, no acceleration model, no
learned perception, no physical localisation.

When localisation is added, it needs the same treatment as perception: a measurement with a
timestamp, an accuracy figure, and an explicit "not known" state. A position estimate that is
silently wrong is worse than no position.

## Verification

```bash
./.venv/Scripts/python.exe -m pytest -q
./.venv/Scripts/ruff.exe check .
node --check taxi/static/app.js   # if browser JavaScript changed
```

## Reference

- `references/pipeline.md` — the perception seam in detail: queue semantics, freshness rules, backend contract.
- `docs/ARCHITECTURE-v2.md` §3 — the Tier 1 pipeline.
- `docs/ROADMAP.md` — M5 perception and localisation.
