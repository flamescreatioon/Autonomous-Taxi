# The perception seam


> **Status (2026-10-28).** The invariants below are the rules the implementation must satisfy.
> The **enforcement tests named here do not exist yet** — `tests/test_safety_invariants.py`,
> `tests/test_protocol_freshness.py` and the other files referenced are to be written along with
> the v2 modules. The rules themselves are binding regardless of when the tests arrive.

How a frame becomes an obstacle, and where the freshness guarantee lives.

---

## The pipeline

```text
camera → LatestFrameQueue → PerceptionPipeline → PerceptionBackend → Observation
                                                       ↓
                                            OccupancyGrid → VFH → Controller
```

Three modules, one rule each:

| Module | Rule |
|---|---|
| `taxi/perception/pipeline.py::LatestFrameQueue` | Hold one frame. Drop the previous. Count the drops. |
| `taxi/perception/pipeline.py::PerceptionPipeline` | Discard frames older than `max_age_s` **before** they reach the backend. |
| `taxi/planning/occupancy.py::OccupancyGrid` | Out-of-grid points are occupied. A `None` distance marks nothing. |

---

## `LatestFrameQueue`

```python
from taxi.perception.pipeline import LatestFrameQueue

queue = LatestFrameQueue()
queue.put(frame_a)
queue.put(frame_b)   # frame_a is dropped, not queued
latest = queue.get() # returns frame_b
print(queue.dropped) # 1
```

Why a single slot: **a late frame is worse than no frame.** If the detector is busy for 80 ms and
the camera produces 30 frames in that time, a FIFO queue hands the detector a 2.6-second-old
image. The system would then steer based on where the world was. Dropping keeps the detector
working on the newest available frame, which is the only one worth working on.

The drop count is not an error — it is a health metric. A sustained non-zero drop rate means the
detector cannot keep up, and that must be visible rather than silently degrading the output.

---

## `PerceptionPipeline`

```python
from taxi.perception.pipeline import PerceptionPipeline
from taxi.perception.detectors import NullBackend

pipeline = PerceptionPipeline(NullBackend(), max_age_s=0.2)
observation = pipeline.step(frame)
```

`step()` checks the frame's age against `max_age_s` and returns `None` for a stale frame rather
than delivering it. The backend never sees stale data.

The age is computed from an **injected clock**, not `time.time()` directly, so the freshness rule
is testable. A clock that jumps backwards makes a frame look fresh; the pipeline treats a
backwards jump as stale.

---

## The backend contract

```python
class PerceptionBackend(Protocol):
    def detect(self, frame: Frame) -> Observation: ...
```

`Frame` carries the image, a monotonic timestamp and a sequence number. `Observation` carries
`Detection` objects, each with a class label, a confidence and a bounding box.

### `NullBackend`

The only enabled backend. It returns an empty observation — **no detections**. It is a
placeholder, and it is not a working detector. Anything consuming it must handle the
"no evidence" case correctly.

### `OpenCVBackend`

A stub that raises `NotImplementedError`. It is deliberately not a silent no-op: a caller that
forgets to select a real backend should fail loudly rather than quietly produce empty
observations that look like a clear road.

**No weights are bundled and none are downloaded at runtime.** `*.pt` and `*.onnx` are gitignored.
Adding a detector means adding the weights deliberately, with their provenance recorded, and
updating this section.

---

## From detections to an occupancy grid

```python
from taxi.planning.occupancy import OccupancyGrid, SensorReading

grid = OccupancyGrid(size_m=40.0, resolution_m=0.2)

# A valid reading marks cells along the ray.
grid.mark_sensor_reading(SensorReading(bearing_rad=0.0, distance_cm=142))

# A missing reading marks NOTHING.
grid.mark_sensor_reading(SensorReading(bearing_rad=0.5, distance_cm=None))
```

`distance_cm=None` means "no evidence". It does **not** mean the path ahead is free. This
distinction is the difference between a vehicle that stops for an unknown and one that drives
into something it never saw.

The grid treats out-of-bounds points as occupied. The planner must not reason about free space it
cannot see — an obstacle just outside the grid is still an obstacle.

---

## VFH

```python
from taxi.planning.vfh import compute_candidate_direction

direction = compute_candidate_direction(grid, target_bearing_rad=0.0)
if direction is None:
    ...  # every candidate is blocked — do not pick the least-bad one
```

`None` is a real result and a feature. The planner must be able to say "I have no valid heading".
Returning the least-bad candidate instead removes the only signal that the path is blocked, and
the vehicle drives into the wall at a slightly smaller angle.

---

## Controller

```python
from taxi.planning.controller import Controller

controller = Controller(max_linear=0.8, max_angular=1.5)
linear, angular = controller.compute(
    current_bearing_rad=0.0,
    target_bearing_rad=1.2,
    dt_s=0.05,
)
```

Acceleration is clamped, and a large heading error reduces linear speed rather than being
satisfied entirely by angular velocity. `dt_s <= 0` raises `ValueError` — a non-positive timestep
means the caller has a bug, and silently returning a zero or a garbage command would hide it.

Velocities are **normalised** to `[-1.0, 1.0]`, not metres per second. The physical scale depends
on a drivetrain that has not been confirmed.

---

## What still does not exist

- Real detector weights or a working detector.
- Camera integration into the pipeline.
- Localisation or position tracking.
- Depth estimation.

Monocular depth, when added, is a **planning hint**, never a safety input. The emergency reflex
uses sonar, and a confidence-scaled detection output is not a stopping distance.
