# Hardware confirmation checklist


> **Status (2026-10-28).** The invariants below are the rules the implementation must satisfy.
> The **enforcement tests named here do not exist yet** — `tests/test_safety_invariants.py`,
> `tests/test_protocol_freshness.py` and the other files referenced are to be written along with
> the v2 modules. The rules themselves are binding regardless of when the tests arrive.

Every item below must be **recorded in `docs/HARDWARE.md`** before any physical output is
enabled. "Probably" is not a value. "Unconfirmed" is an acceptable value; a guess is not.

---

## Before any motor work

### Controller board

| Fact | Value | Why it matters |
|---|---|---|
| Exact module (not the family) | | Pin maps and peripherals differ between variants that share a name. |
| PSRAM variant and size | | The camera buffer count and resolution depend on it. |
| Camera sensor and its lens FOV | | Determines the calibration the planner needs. |
| Camera pin map | | The ESP32-S3 devkit default map does **not** apply to every module. |
| Free GPIO after camera, SD, USB and boot pins | | The camera pin map is not free for anything else. |

### UART link

| Fact | Value | Why it matters |
|---|---|---|
| ESP32 UART TX and RX pins | | To be `-1` in `firmware/esp32_s3/src/main.cpp` (target not yet implemented). |
| Arduino UART pins | | Must not collide with USB serial on an Uno (pins 0/1). |
| Logic levels | | The Arduino's 5 V TX into a 3.3 V ESP32 needs level shifting. |
| Baud rate | | 115200 in the protocol. |
| Cable length / type | | A long run needs differential signalling or a buffer, not a jumper. |

### Driver and drivetrain

| Fact | Value | Why it matters |
|---|---|---|
| Driver part number | | L298N and TB6612FNG have different pin functions, logic levels and current handling. |
| Motor voltage and stall current | | Sets the supply and the protection needed. |
| Motor type | | The kinematic solver assumes a **differential** platform. A steering-servo platform needs a different solver entirely. |
| Encoder type and count per revolution | | Required for closed-loop speed control and for measured stopping distance. |
| Wheelbase / track width | | Needed to convert `(v, ω)` into a turning radius. |
| Direction convention | | Which wheel spins faster for a left turn. |

### Sensors

| Fact | Value | Why it matters |
|---|---|---|
| Ultrasonic model | | HC-SR04's 5 V echo signal needs level shifting into the ESP32; the Arduino reads it directly. |
| Mount positions and angles | | The emergency thresholds depend on what the sensor is actually looking at. |
| IMU model, if used | | |
| Measured trigger-to-echo calibration | | The distance conversion is only as good as this. |

### Power and safety

| Fact | Value | Why it matters |
|---|---|---|
| Battery / supply voltage and capacity | | |
| Separate motor supply | | Motors must never be powered from a GPIO rail. |
| Common signal reference | | Both boards need a shared ground. |
| Emergency stop wiring | | Must be **independent hardware**. No protocol, service or software state can inhibit it. |
| Brake behaviour on disable | | Does the vehicle coast, or does it short-brake? This determines the stopping distance. |

---

## The three-tier measurement checklist

Before trusting the architecture, these must be measured on the real hardware. None of them has
been measured.

| Measurement | How | Feeds |
|---|---|---|
| Camera capture → JPEG available | Timestamps at capture and at encode | The latency budget |
| JPEG encode time | Same, minus capture | The latency budget |
| Wi-Fi uplink latency, campus network | Repeated pings with the real payload size | The latency budget |
| Inference time, real weights | Time the actual detector on the actual frame | The latency budget |
| UART round trip | Loop a frame back and time it | The latency budget |
| **Total end-to-end** | All of the above, summed with real measurement | The 120 ms target |
| Sonar trigger → echo, at range | Distance-tape comparison | The distance conversion |
| **Stopping distance** | From command cut to standstill, with the real vehicle | The emergency thresholds |
| **Watchdog halt time** | Cut the link and measure when motion stops | The 300 ms timeout |
| False sonar echoes | Observe in a realistic environment | The echo timeout |

The emergency thresholds (**20 cm centre, 15 cm sides**) and the watchdog timeout (**300 ms**)
are **provisional design values**. They must be re-derived from these measurements before they
become physical limits. See `docs/adr/ADR-0006.md`.

---

## Recording format

Update `docs/HARDWARE.md` in place. For each item, record either the confirmed value or
`Unconfirmed`. When a value is confirmed, note **how** it was confirmed (datasheet, bench
measurement, continuity test) and **when**.

A row that says a value with no confirmation method is not a confirmed value — it is a guess with
a table around it.
