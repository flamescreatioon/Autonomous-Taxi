# Protocol v2 — binary framing, motion and telemetry

**Status:** **Specification only.** The framing, motion, telemetry and transport layers described
here are **not implemented** in this repository. There is no `taxi/protocol/` package, and neither
board implements the binary framing. What the running code implements today is the newline-ASCII
bench protocol described in [PROTOCOL.md](PROTOCOL.md). No board is connected.

**Related decisions:** [ADR-0003](adr/ADR-0003.md) (binary framing) · [ADR-0004](adr/ADR-0004.md)
(freshness) · [ADR-0005](adr/ADR-0005.md) (velocity scale)
**Related:** [ARCHITECTURE-v2.md](ARCHITECTURE-v2.md) · [PROTOCOL.md](PROTOCOL.md) (the v0.1 bench
protocol that is still what the running code implements)

---

## 0. Corrections to the original draft

The three-tier proposal contained four defects in its protocol section. They are recorded here
because the byte layouts below are the ones to be implemented, and any document still quoting
the draft is wrong.

| # | Draft said | Problem | Corrected specification |
|---|---|---|---|
| 1 | Downstream command is a "fixed 8-byte" packet, with 7 bytes of content listed | Neither 7 nor 8 bytes can carry a session id, a sequence number and two velocities | **16 bytes** — `NET_COMMAND_SIZE` |
| 2 | Network velocities are `int16 ±1000`; UART velocities are `int8 ±100` | Two different meanings for the same quantity | One canonical scale (`MOTION_SCALE = 1000`) with an explicit boundary conversion |
| 3 | A heartbeat renews command authority | A queued old command would acquire a fresh lifetime on late arrival | Freshness is end-to-end via session id + monotonic sequence + timestamp age ([ADR-0004](adr/ADR-0004.md)) |
| 4 | Video header is "bytes 0–3 sequence, 4–7 timestamp" | The upstream video header is separate from the downstream command frame, which was underspecified | Video header is documented independently in §5; the command frame has its own explicit layout |

---

## 1. Conventions

- **Byte order:** little-endian for all multi-byte integers.
- **Signedness:** velocities are signed two's-complement; distances and distances-as-bytes are
  unsigned.
- **Ranges:** every numeric field has an explicit valid range. Values outside it are invalid, not
  clamped into meaning.
- **Units:** velocities are **normalised** — `1.0` means "full commanded magnitude". They are not
  metres per second, because the physical scale depends on a drivetrain that has not been
  confirmed. Distances are **centimetres**. Times are **milliseconds**.

---

## 2. UART framing — COBS + CRC-8

Used for the ESP32-S3 ↔ Arduino link. To be implemented in `taxi/protocol/framing.py` and mirrored
in `firmware/esp32_s3/src/framing.{h,cpp}`.

### 2.1 Frame structure

```text
+--------+--------+------------------+--------+
| 0xAA   | COBS   | COBS payload     | 0x55   |
| delim  | block  | (includes CRC-8) | term   |
+--------+--------+------------------+--------+
```

| Field | Value | Meaning |
|---|---|---|
| Delimiter | `0xAA` | Start of frame; the decoder resynchronises here |
| COBS block | variable | Zero-free encoding of the payload |
| Terminator | `0x55` | End of frame |

The payload **before** COBS encoding is:

```text
+---------+------------------+---------+
| body    | CRC-8            | body    |
| (n B)   | (1 B)            | length  |
+---------+------------------+---------+
```

The CRC covers the body only. It is placed between the body and the trailing length byte so that
a decoder can locate it without knowing the body length in advance.

### 2.2 COBS

Consistent Overhead Byte Stuffing. The encoding guarantees the output contains **no zero bytes**,
which is what makes byte-oriented delimiter scanning reliable.

The implementation is a faithful port of the reference `cobs` library algorithm. The important
subtlety is the **zeros-preserved** variant: the code byte is emitted at every zero boundary and
at the end of the payload, and the decoder appends a zero only when
`index < len(encoded) and length < 0xFF`. A decoder that appends the zero unconditionally — the
more commonly seen simplification — will disagree at the final block.

### 2.3 CRC-8

| Parameter | Value |
|---|---|
| Width | 8 bits |
| Polynomial | `0x07` (CRC-8/ATM, reflected) |
| Initial value | `0x00` |
| Input reflected | No |
| Output reflected | No |
| Final XOR | `0x00` |
| Check value for `"123456789"` | **`0xF4`** |

The polynomial and initial value are stated explicitly rather than by name alone, because
"CRC-8" names at least a dozen incompatible variants.

### 2.4 Bounds and error handling

| Limit | Value | On violation |
|---|---|---|
| Maximum frame length | 64 bytes (`MAX_FRAME_LENGTH`) | `FrameTooLong`; the frame is discarded |
| Maximum payload | 62 bytes (64 − delimiter − terminator) | same |
| Bad CRC | — | `BadChecksum`; the frame is discarded |
| Malformed COBS | — | `BadHeader`; the decoder resynchronises on the next `0xAA` |
| Missing terminator | — | the frame stays incomplete until the next delimiter arrives |

`FrameDecoder` is an **incremental** decoder: it accepts arbitrary byte chunks, buffers partial
frames, and on a bad frame discards up to the next delimiter rather than discarding the whole
buffer. `rejected_frames` counts the discards, so link quality is observable.

### 2.5 Cross-language guarantee

To be enforced by a shared-vector test that compiles `firmware/esp32_s3/src/framing.cpp` and
asserts that the C++ and Python implementations produce **identical bytes** for the same set of
input vectors, and that their CRC-8 implementations agree. The test skips — visibly — when no C++
compiler is available.

This is to be the highest-value test in the v2 suite. A framing mismatch between the host and the
firmware is silent, intermittent, and dangerous.

---

## 3. Motion command — the canonical domain

To be implemented in `taxi/protocol/motion.py`.

### 3.1 The canonical scale

| Constant | Value | Meaning |
|---|---|---|
| `MOTION_SCALE` | `1000` | `1000` = full scale (`1.0`) |
| `PROTOCOL_VERSION` | `1` | Protocol version carried in every motion frame |
| `DEFAULT_MAX_AGE_S` | `0.3` | Maximum accepted command age, in seconds |

Velocities on the wire are integers in `[-1000, +1000]`. The normalised form is
`value / MOTION_SCALE`, in `[-1.0, +1.0]`.

`MotionCommand.create()` takes normalised floats and **clamps** out-of-range values rather than
wrapping them, so a planner bug cannot produce an unexpectedly large command.

### 3.2 The command frame

```text
MotionCommand (conceptual)
  version       uint8    protocol version
  session_id    uint32   changes whenever motion authority is re-established
  sequence      uint32   strictly increasing within a session
  linear        int16    canonical scale, [-1000, +1000]
  angular       int16    canonical scale, [-1000, +1000]
  timestamp_ms  uint32   sender's monotonic clock
```

`session_id` is the mechanism that makes a reconnect safe. A command queued before a reconnect
belongs to the old session and is discarded on arrival. `sequence` rejects reordering and replay
within a session. `timestamp_ms` rejects commands that are simply too old.

### 3.3 Freshness rule

`is_fresh(command, now_ms, max_age_s, clock_uncertainty_ms)` is to implement the acceptance test:

1. `age_ms = now_ms - command.timestamp_ms`
2. If `age_ms < -clock_uncertainty_ms` → **reject**. The command is future-dated beyond the stated
   clock uncertainty. A future timestamp cannot be trusted to age out correctly, so it is treated
   as unfresh rather than as infinitely fresh.
3. If `age_ms > max_age_s × 1000 + clock_uncertainty_ms` → **reject**. Too old.
4. Otherwise → accept.

Note the direction of the uncertainty term. Clock uncertainty **widens** the acceptance window in
both directions: it tolerates a slightly-old timestamp and a slightly-future one. It never
extends a command's effective lifetime past the age limit in the "make it live longer" sense —
that is governed by `max_age_s` alone.

The **sequence** and **session** checks sit alongside this and are enforced by the receiver:

- A `sequence` that is not strictly greater than the last accepted one is rejected (replay and
  reordering).
- A `session_id` that does not match the current session is rejected, and pending commands from
  the previous session are flushed.

**A heartbeat does not renew motion authority.** A ping is not a motion command and carries no
velocity. Only an accepted, fresh, correctly-sequenced motion command advances the watchdog.

### 3.4 Session establishment

Motion authority is established by the sender choosing a **new** `session_id` and then sending
motion commands with strictly increasing `sequence` numbers. On the Arduino side:

- The watchdog starts **expired** after boot or reset. The first accepted command rearms it.
- A session change flushes any pending commands before the new session's first command is applied.

A reconnect therefore always starts from "no motion" and must be re-established deliberately.

---

## 4. Packet layouts

### 4.1 Server → ESP32 — network command (16 bytes)

To be implemented in `taxi.protocol.transport.NetworkCommand`.

```text
Offset  Size  Field         Value / range
------  ----  ------------  -----------------------------------------------
0       1     magic         0x5A
1       1     packet_type   0x01 motion | 0x02 ping | 0x03 telemetry request
2       1     version       1
3       1     reserved      0
4       4     session_id    uint32 LE
8       4     sequence      uint32 LE, strictly increasing within the session
12      2     linear        int16 LE, [-1000, +1000]
14      2     angular       int16 LE, [-1000, +1000]
```

A trailing marker `0xA5` is reserved in `NET_COMMAND_TRAILER` for a future frame variant; it is
**not** part of the current 16-byte layout.

The draft's "fixed 8-byte" description cannot hold these fields. 16 bytes is the smallest frame
that carries session id, sequence and two 16-bit velocities.

### 4.2 ESP32 → Arduino — UART motion packet (18 bytes)

To be implemented in `taxi.protocol.transport.UartMotionPacket`.

```text
Offset  Size  Field         Value / range
------  ----  ------------  -----------------------------------------------
0       1     packet_type   0x01 motion
1       1     version       1
2       4     session_id    uint32 LE
6       4     sequence      uint32 LE
10      2     linear        int16 LE, canonical scale
12      2     angular       int16 LE, canonical scale
14      4     timestamp_ms  uint32 LE
```

This is the **payload**; on the wire it is COBS-encoded, CRC-8 protected and wrapped in the
`0xAA` / `0x55` frame of §2.

### 4.3 The UART scale boundary

The draft used `int8 ±100` on the UART and `int16 ±1000` on the network. That is not a conflict of
units — it is one canonical domain expressed at two resolutions, with the conversion made explicit
at the boundary rather than left implicit:

```python
normalise_for_uart(value) = clamp(round(value × 127 / MOTION_SCALE), -127, +127)
expand_from_uart(value)    = round(value × MOTION_SCALE / 127)
```

**The implemented `UartMotionPacket` uses the `int16` canonical form** (offsets 10–14), not the
`int8` form — the packet is 18 bytes precisely because both velocities are carried at full
canonical width. The conversion helpers exist for the reduced-width case and are covered by
tests, but they are **not on the path the current packet layout takes**. If the UART frame is
later narrowed to `int8`, this is where the conversion must be applied, and the change must be
made on both sides of the link.

The canonical value is always the `±1000` one. `expand_from_uart(normalise_for_uart(x)) == x` for
every representable canonical value.

### 4.4 Arduino → ESP32 — UART telemetry packet (11 bytes)

To be implemented in `taxi.protocol.transport.UartTelemetryPacket`.

```text
Offset  Size  Field         Value / range
------  ----  ------------  -----------------------------------------------
0       1     packet_type   0x03 telemetry
1       1     left_cm       uint8, centimetres, 0xFF = invalid
2       1     center_cm     uint8, centimetres, 0xFF = invalid
3       1     right_cm      uint8, centimetres, 0xFF = invalid
4       1     state_bits    see below
5       2     uptime_ms     uint16 LE, wraps
7       4     age_ms        uint32 LE, age of the reported distances
```

**State bits** (to be implemented in `taxi/protocol/telemetry.py`):

| Bit | Mask | Meaning |
|---|---|---|
| 0 | `0x01` | Emergency stop active |
| 1 | `0x02` | Watchdog tripped |
| 2 | `0x04` | Motors enabled |
| — | `0xFF` | `DISTANCE_INVALID` sentinel for an individual distance |

`age_ms` is the Arduino's own measurement of how old the reported distances are. It lets the host
distinguish a fresh reading from a held-over one — a distinction the host cannot make from the
distance bytes alone.

### 4.5 Telemetry validity

`TelemetryFrame.distances_valid` is true only when **all three** distances are valid. A single
invalid distance makes the frame invalid as a whole for the purposes of the emergency reflex,
because "no information about the left side" is not "the left side is clear".

---

## 5. Video and host-side telemetry (WebSocket)

### 5.1 Upstream binary video

```text
Offset  Size  Field
------  ----  ------------------
0       4     sequence       uint32 LE
4       8     timestamp_ms    uint64 LE
8       N     JPEG payload    remainder of the frame
```

The header is 8 bytes and the JPEG payload is the rest of the frame. The draft's description
("bytes 0..3 sequence, 4..7 timestamp") is consistent with this for the *video* header only —
the defect was that the draft applied an 8-byte size to the *downstream command*, which is a
different frame with different contents. The two headers are independent.

The JPEG is produced on the ESP32-S3 with `PIXFORMAT_JPEG`, QVGA or VGA, `jpeg_quality=12`, into
PSRAM with two frame buffers (`fb_count=2`).

### 5.2 Upstream JSON telemetry at 10 Hz

Telemetry is published at 10 Hz. Every value carries its **source**, **age** and **validity**:

```json
{
  "source": "simulated",
  "age_s": 0.04,
  "valid": true,
  "values": { "center_cm": 142, "left_cm": 88, "right_cm": 91 },
  "reason": null
}
```

`source` distinguishes `"simulated"` from a device source. `valid=false` always carries a `reason`
and **empty** values. The host-side rule is explicit:

> **Missing or stale sensors are invalid, never substituted with fabricated healthy values.**

`taxi/comms/health.py::assess()` is to implement the shared rules: a sample is invalid when it is
stale, when a required key is missing, or when the source reports `sensors_ready=false`.

### 5.3 Downstream

The 16-byte network command of §4.1 is the only motion-bearing frame in this direction.

---

## 6. Arduino state machine constants

From `firmware/arduino/src/`:

| Constant | Value | Source | Physical? |
|---|---|---|---|
| `kWatchdogTimeoutMs` | 300 ms | `firmware/arduino/src/watchdog.h` (to be written) | **Provisional — needs measurement** |
| `kEmergencyCenterCm` | 20 cm | `firmware/arduino/src/arbiter.h` (to be written) | **Provisional — needs measurement** |
| `kEmergencySideCm` | 15 cm | `firmware/arduino/src/arbiter.h` (to be written) | **Provisional — needs measurement** |
| `kSonarPhaseIntervalMs` | 25 ms | `firmware/arduino/src/sonar_fsm.h` (to be written) | Provisional |
| `kSonarEchoTimeoutMs` | 5 ms | `firmware/arduino/src/sonar_fsm.h` (to be written) | Provisional |
| `kSonarInvalidCm` | -1 | `firmware/arduino/src/sonar_fsm.h` (to be written) | Sentinel, not a physical value |

None of these constants exists in code today — no firmware header defines them. They are recorded
here as the values the implementation is to use, pending measurement.

These values are carried over from the design proposal and have **not** been measured against the
real vehicle. See [ADR-0006](adr/ADR-0006.md) and [HARDWARE.md](HARDWARE.md).

---

## 7. What is actually wired, and what is not

Honest accounting of the implementation state of each layer. **Nothing in the v2 protocol is
implemented in this repository**; the right-hand column states what exists today.

| Layer | State |
|---|---|
| `taxi/protocol/framing.py` | **Not implemented.** No `taxi/protocol/` package exists. |
| `firmware/esp32_s3/src/framing.{h,cpp}` | **Not implemented.** No `firmware/esp32_s3/` directory exists. |
| `taxi/protocol/{motion,telemetry,transport}.py` | **Not implemented.** Packet layouts, freshness rules and scale conversion are specified above and to be written. |
| `firmware/esp32_s3/src/protocol.h` | **Not implemented.** The mirrored constants are to be written with the target. |
| `firmware/arduino/src/` framing | **Not implemented.** The Arduino has no framing module. Its `main.cpp` still parses the newline-ASCII bench protocol only. |
| ESP32-S3 → Arduino binary link | **Not wired.** Neither end of the binary link exists. |
| `NetworkCommand` timestamp | **Not carried.** The 16-byte network frame has no room for it, so the timestamp is carried on the UART motion packet, where the freshness check actually happens. |
| Newline-ASCII bench protocol | **Implemented and running** — `firmware/arduino/src/main.cpp` and `firmware/esp32_cam/src/main.cpp`. See [PROTOCOL.md](PROTOCOL.md). |

Two consequences worth stating plainly:

1. **The Arduino's binary receive path is entirely future work.** The `UartMotionPacket` layout in
   §4.2 is specified here, but no code encodes or decodes it. Until it does, the Arduino accepts
   only `PING` and `STOP`.
2. **The end-to-end freshness check has no implementation at all.** The rule in §3.3 is specified
   here, and the session and sequence fields are present in the packet layouts, but no host code
   evaluates it and no board applies it. This is the single largest gap between the specified
   protocol and the running firmware.

### The v0.1 bench protocol is still what runs

The newline-ASCII protocol in [PROTOCOL.md](PROTOCOL.md) — `PING\n`, `STOP\n`, JSON telemetry over
USB serial — **is what the current bench firmware implements and accepts.**

The two are to coexist deliberately: the bench firmware keeps working while the binary stack is
brought up beside it. The transition is a hardware-confirmation milestone, not a code-complete
milestone.

---

## 8. Test coverage for this protocol

> **The tests below are to be written.** None of them exists yet. The current suite is
> `tests/test_api.py` and `tests/test_supervisor.py` only.

| File (to be created) | Covers |
|---|---|
| `tests/test_protocol_framing.py` | CRC-8 check value, round-trip, bad CRC/header/terminator, oversized frames, split-frame reassembly, resynchronisation after corruption |
| `tests/test_protocol_freshness.py` | Scale round-trip, clamping, fresh/expired/future acceptance, clock-uncertainty widening, late-arrival rejection, replay rejection |
| `tests/test_comms.py` | Simulated and device sources, invalid-on-failure, `assess()` staleness and missing-key rules |
| `tests/test_cross_language_framing.py` | Python ↔ C++ byte-for-byte agreement on encoded frames and on CRC-8 |
| `tests/test_safety_invariants.py` | Protocol modules contain no I/O side effects; firmware pins remain unset |

---

## 9. Sources

- COBS reference implementation: the `cobs` PyPI package, `cobs/cobs/_cobs_py.py`
- CRC-8/ATM: ITU-T Rec. I.432 (identical parameters)
- ESP32 camera API: https://github.com/espressif/esp32-camera
