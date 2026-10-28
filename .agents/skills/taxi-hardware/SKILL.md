---
name: taxi-hardware
description: Use this skill for any work on the Autonomous Taxi's physical boards and links - ESP32-S3, ESP32-CAM, Arduino Uno, PlatformIO, pio run, UART, COBS framing, CRC-8, camera pin map, HC-SR04 sonar, motor driver, L298N, TB6612FNG, watchdog, arbiter, kinematics, PWM, flashing, bench testing, wiring, HARDWARE.md, secrets.h, device token, jpeg_quality, fb_count, PSRAM, FreeRTOS task pinning. Trigger before writing ANY line of firmware/ code, before enabling any output, before claiming a build or a hardware test succeeded, and whenever someone asks to wire a motor, read a sensor, or flash a board.
---

# Taxi Hardware — ESP32-S3, Arduino, and the bench rules

> **Repository state (2026-10-28).** The three-tier architecture these skills describe is
> **specified but not implemented**. There is no `taxi/protocol/`, `taxi/comms/`,
> `taxi/perception/`, `taxi/planning/` or `taxi/mission/` package, no `firmware/esp32_s3/`
> target, and no binary framing code on either board. What exists is the v0.1 simulation, the
> ESP32-CAM bench starter and the Arduino newline-ASCII bench parser. Treat every path and
> constant named below as **the target design**, not as code you can read or run. See
> `docs/VALIDATION.md` for the verified state of the current tree.


Everything here is currently **bench-only and disarmed**. No physical output is enabled anywhere in
this repository, and a test enforces that.

## The rule that outranks every other instruction in this file

> **Never enable a physical output with a guessed pin, a guessed driver type, or a guessed safety
> threshold.**

If a task would require any of those three, the task is blocked on hardware confirmation, not on
your cleverness. Say so plainly and record what must be confirmed.

Before motor work, `docs/HARDWARE.md` must record the confirmed hardware, and fault-injection
tests must cover the complete host → ESP32 → Arduino path.

## Separate the three claims

When you report on hardware work, state these separately and never blur them:

| Claim | What it means |
|---|---|
| **Target build** | The firmware compiled for a target. `pio run` succeeded. |
| **Hardware test** | The board was flashed, connected, and observed behaving as expected. |
| **Motion validation** | The vehicle moved under control, with stopping measured. |

A successful compile is not a hardware test. A successful hardware test is not motion validation.
The current repo state is: `firmware/arduino` and `firmware/esp32_cam` **compile**; there is
**no hardware test**. The ESP32-S3 target does not exist in this repository.

## Firmware layout

| Path | Target | Notes |
|---|---|---|
| `firmware/esp32_s3/` | `esp32-s3-devkitc-1`, `espressif32@6.9.0` | **Primary target — not yet implemented.** FreeRTOS task layout, PSRAM camera, UART bridge, WS client are specified in `docs/ARCHITECTURE-v2.md`. |
| `firmware/arduino/` | `uno`, `atmelavr@5.1.0` | **Bench parser only.** Newline-ASCII line handling; no sensors, no motors. The 50 Hz safety loop, sonar FSM, arbiter, watchdog and kinematics are specified but not implemented. |
| `firmware/esp32_cam/` | AI Thinker ESP32-CAM | Legacy bench fallback. Still works; still supported. |

The ESP32-CAM is demoted to a *fallback*, not deleted: it is the board actually in hand, and it
keeps the bench path alive while the S3 hardware is confirmed.

## Current disarmed state — preserve it

No firmware in this repository can drive a motor: there is no actuator GPIO, no driver
integration, and no sensor code at all. A guardrail test (`tests/test_safety_invariants.py`, to be
written alongside the v2 firmware) is to fail if that ever changes. Do not "fix" such a test.

- **`firmware/esp32_s3/src/main.cpp`** (to be written): every `kCameraPin*`, `kUartTxPin` and
  `kUartRxPin` is to be `-1`.
- **`firmware/arduino/src/main.cpp`**: contains no `analogWrite`, no `digitalWrite`, no `MOTOR_PIN`.
  The echo callbacks `onTrigger`/`onCapture` are empty, so every sonar reading stays invalid.
- The firmware reports `motion_enabled=false` and `sensors_ready=false`.

If you are asked to enable outputs, the correct action is to fill in `docs/HARDWARE.md` first and
get the hardware facts confirmed — not to assign a pin.

## The three-tier safety model

Safety authority flows **downward only**. The laptop may request motion; only the Arduino grants
it. See `docs/ARCHITECTURE-v2.md`.

**Tier 2 — ESP32-S3.** FreeRTOS, dual cores:

| Core | Task | Priority |
|---|---|---|
| 0 | `Task_NetworkClient` | 4 |
| 0 | `Task_CommandRelay` | 3 |
| 1 | `Task_CameraCapture` | 5 |
| 1 | `Task_TelemetryPoll` | 2 |

Camera target: PSRAM, `fb_count=2`, `PIXFORMAT_JPEG`, QVGA/VGA, `jpeg_quality=12`.

**Tier 3 — Arduino Uno.** 50 Hz (20 ms) cyclic loop, non-blocking. **No `delay()` and no
`pulseIn()`** in the control path — `pulseIn()` blocks for the duration of the measurement and
would stall the watchdog, the arbiter and the actuator update together. Echo capture is
interrupt-driven.

Sonar is a 3-phase round-robin: centre at 0 ms, left at 25 ms, right at 50 ms. A timed-out echo is
invalid, not zero.

The arbiter tiers, in priority order:

| Priority | Tier | Condition |
|---|---|---|
| 0 | Fail-safe | Watchdog expired (> 300 ms since last accepted command) |
| 1 | Emergency reflex | Centre < 20 cm, or side < 15 cm, **and the reading is valid** |
| 2 | Nominal | Valid fresh command and valid sensors |

A missing sensor does **not** trigger the reflex — "no reading" is not "reading zero".

The threshold values above (300 ms, 20 cm, 15 cm) are **provisional**, carried over from the design
proposal. They have not been measured. See `docs/adr/ADR-0006.md`.

## Framing contract

The UART link is to use COBS framing with CRC-8, and the C++ implementation must match the Python
one **byte for byte**. `tests/test_cross_language_framing.py` (to be written) compiles the C++ and
asserts identical bytes for shared vectors — do not change one side without the other. Neither
implementation exists yet; both are specified in `docs/PROTOCOL-v2.md` §2.

| Parameter | Value |
|---|---|
| Delimiter / terminator | `0xAA` / `0x55` |
| CRC-8 | poly `0x07`, init `0x00` (CRC-8/ATM), check value for `"123456789"` = `0xF4` |
| Max frame | 64 bytes |

If you touch `firmware/esp32_s3/src/framing.{h,cpp}` (to be written), the Python mirror is
`taxi/protocol/framing.py` (also to be written), and the test compares them.

## Build and flash

```bash
cd firmware/esp32_s3
pio run            # build only
pio run -t upload  # flash
pio device monitor
```

The PlatformIO toolchain must be downloaded on first use. If the download fails or is blocked,
**report that plainly** — do not claim a successful compile.

Before an ESP32-CAM build, copy `src/config.example.h` to `src/secrets.h` and set Wi-Fi
credentials and a long device token. `secrets.h` is gitignored. Never put a token in a URL.

## Verification

```bash
./.venv/Scripts/python.exe -m pytest -q     # includes the cross-language framing test
./.venv/Scripts/ruff.exe check .
```

Report which of these actually ran, and their exact outcome.

## Reference

- `references/hardware-checklist.md` — every fact that must be recorded before motor work.
- `references/framing.md` — the framing contract in detail, for when you change it.
- `docs/HARDWARE.md` — the confirmation record.
- `docs/PROTOCOL-v2.md` — byte layouts.
- `docs/adr/ADR-0002.md` (S3 primary), `ADR-0003.md` (framing), `ADR-0004.md` (freshness).
