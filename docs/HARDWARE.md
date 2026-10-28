# Hardware confirmation and bench setup

No hardware is connected to the development environment. The firmware is a starting point, not
hardware-tested vehicle control.

> **Never enable a physical output with a guessed pin, a guessed driver type, or a guessed safety
> threshold.** Every cell in the table below is either a confirmed fact or `Unconfirmed`. A guess
> is not an acceptable entry.

---

## Target boards

| Target | Role | Status |
|---|---|---|
| **ESP32-S3** (`esp32-s3-devkitc-1`) | **Primary edge gateway** (target architecture). Dual-core Xtensa, PSRAM, PSRAM-backed camera DMA, FreeRTOS task layout. | **Not present in this repository.** Specified in [ARCHITECTURE-v2.md](ARCHITECTURE-v2.md); no `firmware/esp32_s3/` directory exists |
| **Arduino Uno R3** (`uno`) | Actuation and safety tier (target architecture). 50 Hz loop, sonar FSM, subsumption arbiter, watchdog, kinematics. | **Bench starter only** — a newline-ASCII line parser. The safety structure is specified in [PROTOCOL-v2.md](PROTOCOL-v2.md) but **not implemented**. **Compiles.** |
| ESP32-CAM (AI Thinker) | **Legacy bench fallback.** Kept supported because it is the board actually in hand. | Existing starter, working source. **Compiles.** |

The ESP32-CAM is demoted to a fallback, not deleted — see
[ADR-0002](adr/ADR-0002.md). It keeps the bench path alive while the S3 hardware is confirmed.

---

## Confirmation table

| Item | Current value | Must confirm |
|---|---|---|
| **Edge board** | ESP32-S3 `esp32-s3-devkitc-1` (assumed) | Exact module and variant — the S3 family has several carrier designs |
| **PSRAM** | Assumed present | Variant and size; buffer count and resolution depend on it |
| **Camera** | ESP32-CAM AI Thinker OV2640 — pin map embedded in `firmware/esp32_cam/src/main.cpp` | Sensor part, lens FOV, full pin map, free GPIO after camera/SD/USB/boot pins |
| **Controller** | Uno (assumed) | Uno, Nano, Mega or another model — the target in `platformio.ini` is `uno` |
| **Driver** | None | L298N, TB6612FNG or another driver — part number, logic levels, current handling |
| **Steering** | None | Differential motors or a steering servo — the target kinematics solver assumes **differential** |
| **Motion feedback** | None | Encoder type and counts per revolution |
| **Mechanics** | None | Wheelbase, track width, motor direction convention |
| **Ultrasonics / IMU** | Not implemented | Models, pins, trigger-to-echo calibration, mount positions and angles |
| **Emergency stop** | Not implemented in firmware | Independent hardware disable, and the resulting braking behaviour |
| **UART** | ESP32-CAM relay pins unset (`-1`); Arduino parses USB serial | RX/TX pins on both boards, level shifting, boot-pin constraints |
| **Power** | None | Supply voltage, capacity, separate motor rail, common signal reference |

---

## Facts that must be recorded before motor work

This is the gate. **All of these must be filled in and reviewed before any physical output is
enabled.**

1. **Exact edge board module** — not the family, the module.
2. **PSRAM variant and size.**
3. **Camera pin map** — every pin, and which pins remain free for other use.
4. **UART TX/RX pins on both boards**, with the level-shifting arrangement, and confirmation that
   no camera, SD, USB or boot pin is in conflict.
5. **Driver part number** and its logic levels and current ratings.
6. **Steering type** — differential or servo. This determines whether the `(v, ω)` solver is even
   correct.
7. **Encoder feedback** — type and counts per revolution.
8. **Independent emergency-stop wiring**, and what the vehicle does when it is asserted (coast or
   short-brake). This determines the stopping distance.
9. **Measured stopping distance** from command cut to standstill.
10. **Measured watchdog halt time** — cut the link, measure when motion stops.

See `docs/adr/ADR-0006.md` for the threshold re-derivation requirement and the checklist in
`.agents/skills/taxi-hardware/references/hardware-checklist.md` for the full measurement list.

---

## Provisional thresholds

These are carried over from the design proposal. **None has been measured, and none is implemented
in this repository** — no firmware source defines them yet. They must be re-derived from bench
measurement before they are treated as physical limits, and simulator constants must never be
promoted into them.

| Constant | Value | Status |
|---|---|---|
| Watchdog timeout | 300 ms | Design value — not implemented, not measured |
| Emergency centre distance | 20 cm | Design value — not implemented, not measured |
| Emergency side distance | 15 cm | Design value — not implemented, not measured |
| Sonar phase interval | 25 ms | Design value — not implemented, not measured |
| Sonar echo timeout | 5 ms | Design value — not implemented, not measured |

The values themselves are specified in [PROTOCOL-v2.md](PROTOCOL-v2.md) and
[adr/ADR-0006.md](adr/ADR-0006.md).

---

## ESP32-CAM bench build (legacy fallback)

Install PlatformIO, open `firmware/esp32_cam`, and copy `src/config.example.h` to
`src/secrets.h`. Set 2.4 GHz Wi-Fi credentials and a long device token. Keep relay pins `-1` until
wiring is confirmed. The AI Thinker camera pin map is embedded in `main.cpp` and must match the
actual module.

```bash
pio run
pio run -t upload
pio device monitor
```

Build versions are specified in `platformio.ini`. The pinned Espressif platform uses the
Arduino-ESP32 2.x camera API. If you upgrade the platform, check camera struct/API compatibility.

**This target compiles** once `src/secrets.h` exists — see [VALIDATION.md](VALIDATION.md). Note
that the Arduino-ESP32 2.x `httpd_err_code_t` enum has no 503 member, so `/stop` reports
`501 Method Not Implemented` when the UART is not configured.

Read the printed IP, then on Windows:

```powershell
$env:ESP32_DEVICE_TOKEN = 'YOUR_DEVICE_TOKEN'
.\.venv\Scripts\python.exe scripts\camera_probe.py http://192.168.1.50
```

Replace the IP with the board's IP. The probe saves `runtime/camera.jpg` and makes no movement
request. Never place the token in the URL.

The ESP32 currently captures one JPEG per request. `/telemetry` returns plain text with the last
unvalidated UART line and its age. Before the first line, `NO_TELEMETRY` is reported; `age=0`
alone does not mean health. `/stop` only acknowledges sending a serial request; it does not
confirm physical stopping.

---

## Arduino bench build

```bash
cd firmware/arduino
pio run
pio run -t upload
pio device monitor
```

The target is `uno` on `atmelavr@5.1.0`, built with `-DTAXI_BENCH_ONLY=1`.
**This target compiles** — see [VALIDATION.md](VALIDATION.md) for the recorded result and size
usage.

The current firmware is a **bench line parser only**: it handles `PING`, `STOP` and JSON telemetry
over USB serial. The sonar FSM, watchdog, subsumption arbiter and kinematics described in the
target architecture are **not implemented** — there are no `sonar_fsm`, `arbiter`, `watchdog` or
`kinematics` modules in `firmware/arduino/src/`, which contains `main.cpp` alone.

Consequently:

- **No motor GPIO is assigned.** No `analogWrite`, no `digitalWrite`, no `MOTOR_PIN`.
- There are no sonar readings at all, so there is no obstacle reflex and no watchdog timeout.
- The board cannot be asked to move, because nothing on it can grant movement.

USB serial is shared with pins 0/1 on an Uno; avoid programmer/relay contention.

---

## Before wiring

Confirm a stable supply, a shared signal reference, UART level shifting when a 5 V Arduino
transmits to the 3.3 V ESP32, and camera/SD/boot-pin conflicts. **Do not power motors from GPIO.**

Test the physical disable with motor power isolated before considering actuation. Adding a driver
requires a board-specific implementation and measured stopping behaviour. **There is no switch
enabling motors in this release** — no firmware in this repository can drive a motor at all.
