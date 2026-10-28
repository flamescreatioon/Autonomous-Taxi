# The framing contract


> **Status (2026-10-28).** The invariants below are the rules the implementation must satisfy.
> The **enforcement tests named here do not exist yet** — `tests/test_safety_invariants.py`,
> `tests/test_protocol_freshness.py` and the other files referenced are to be written along with
> the v2 modules. The rules themselves are binding regardless of when the tests arrive.

The UART link between the ESP32-S3 and the Arduino is to use COBS framing with a CRC-8. The
Python and C++ implementations must agree **byte for byte**, and
`tests/test_cross_language_framing.py` (to be written) is what proves they do. **Neither
implementation exists yet** — this document specifies the contract they are to satisfy.

---

## Frame structure

```text
+--------+--------+------------------+--------+
| 0xAA   | COBS   | COBS payload     | 0x55   |
| delim  | block  | (includes CRC-8) | term   |
+--------+--------+------------------+--------+
```

The payload before COBS encoding:

```text
+---------+------------------+---------+
| body    | CRC-8            | body    |
| (n B)   | (1 B)            | length  |
+---------+------------------+---------+
```

The CRC covers the **body only**. It sits between the body and the trailing length byte so the
decoder can find it without knowing the body length in advance.

---

## Parameters

| Parameter | Value | Where |
|---|---|---|
| Delimiter | `0xAA` | `taxi/protocol/framing.py::DELIMITER` |
| Terminator | `0x55` | `taxi/protocol/framing.py::TERMINATOR` |
| CRC-8 polynomial | `0x07` | `taxi/protocol/framing.py::CRC8_POLYNOMIAL` |
| CRC-8 initial value | `0x00` | `taxi/protocol/framing.py::CRC8_INIT` |
| CRC-8 check value for `"123456789"` | `0xF4` | asserted in the test |
| Max frame length | 64 bytes | `taxi/protocol/framing.py::MAX_FRAME_LENGTH` |
| Max payload | 62 bytes | 64 − delimiter − terminator |

The CRC-8 parameters are CRC-8/ATM. They are stated explicitly rather than by name alone,
because "CRC-8" names at least a dozen incompatible variants.

---

## COBS — the part that goes wrong

Consistent Overhead Byte Stuffing guarantees the encoded output contains **no zero bytes**, which
is what makes byte-oriented delimiter scanning reliable.

The implementation is a faithful port of the reference `cobs` library
(`cobs/cobs/_cobs_py.py`). The subtle part is the **zeros-preserved** variant:

- The encoder emits a code byte at every zero boundary **and** at the end of the payload.
- The decoder reads `end = idx + length - 1`, copies the literals, and appends a zero **only when**
  `idx < len(encoded) and length < 0xFF`.

A decoder that appends the zero unconditionally — the more common simplification — will disagree
at the final block. This was a real, subtle bug during development and is exactly why the
cross-language test exists.

---

## Error handling

| Condition | Behaviour |
|---|---|
| Frame longer than 64 bytes | `FrameTooLong`; the frame is discarded |
| CRC mismatch | `BadChecksum`; the frame is discarded |
| Malformed COBS (zero code byte, run past the end, zero inside a run) | `BadHeader`; resynchronise on the next `0xAA` |
| Terminator missing | The frame stays incomplete until the next delimiter arrives |
| Bytes before the first delimiter | Ignored |

`FrameDecoder` is incremental: it accepts arbitrary byte chunks, buffers partial frames, and on a
bad frame discards only up to the next delimiter rather than the whole buffer. `rejected_frames`
counts discards so link quality is observable.

---

## The matching requirement

| Side | File |
|---|---|
| Python | `taxi/protocol/framing.py` (to be written) |
| C++ | `firmware/esp32_s3/src/framing.h` / `framing.cpp` (to be written) |

`tests/test_cross_language_framing.py` (to be written) compiles the C++ and asserts that both implementations
produce **identical bytes** for the same input vectors, and that their CRC-8 implementations
agree. The test skips — visibly — when no C++ compiler is available.

**If you change one side, you must change the other.** The test will tell you if you did not.
A framing mismatch between the host and the firmware is silent, intermittent, and dangerous:
it will work perfectly on the bench and fail intermittently in the field.

### Header ordering constraint in the C++ file

In `firmware/esp32_s3/src/framing.h` (to be written), the free-function declarations `Crc8`, `EncodeFrame` and
`DecodeFrame` **must** appear before the templated `FrameDecoder` class. The class calls
`DecodeFrame` in its template body, and C++ requires the declaration to be visible at template
definition time, not merely at instantiation. Getting this wrong produces:

```
there are no arguments to 'DecodeFrame' that depend on a template parameter
```

which is a confusing error for what is really an ordering problem.
