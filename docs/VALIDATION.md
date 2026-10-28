# Validation record — 28 October 2026

This record covers **v0.1 only**: the committed simulation, API, dashboard and the two
bench firmware starters. The three-tier target architecture is specified in
[ARCHITECTURE-v2.md](ARCHITECTURE-v2.md) and [PROTOCOL-v2.md](PROTOCOL-v2.md), but **none of
it is implemented in this repository yet** — there is no `taxi/protocol/`, `taxi/comms/`,
`taxi/perception/`, `taxi/planning/` or `taxi/mission/` package, no `firmware/esp32_s3/` target,
and no binary framing code on either board.

Anything described in this file is a **verified claim about the current tree**. Statements about
the v2 target appear only in the v2 documents and are labelled there as design, not as code.

## Passed in the development environment

- **14 pytest cases pass** — the complete committed suite (`tests/test_api.py`,
  `tests/test_supervisor.py`). This was verified by running:

  ```powershell
  mkdir -p runtime/pytest-tmp
  .\.venv\Scripts\python.exe -m pytest -q --basetemp=runtime/pytest-tmp
  ```

  Result: `14 passed, 1 warning in 1.03s`.

  **The `--basetemp` flag is required on this machine.** `tests/test_supervisor.py` uses
  `tmp_path`, and pytest's per-user temp root
  (`C:\Users\ADMIN\AppData\Local\Temp\pytest-of-ADMIN`) raises `PermissionError [WinError 5]`.
  Running plain `pytest -q` therefore reports **`13 passed, 1 error`** — that error is the
  environment's broken temp directory, not a defect in the application. `runtime/` is already
  covered by `.gitignore`, so the workaround leaves no untracked files.

  The one warning is a Starlette deprecation notice about `httpx` in `fastapi.testclient`; it
  does not affect the results.

- **Ruff lint check clean** — `.\.venv\Scripts\ruff.exe check .` reports `All checks passed!`.
- **Node JavaScript syntax check** of the dashboard code — `node --check taxi/static/app.js`
  passes. This checks syntax only; it is not browser QA.
- **Both firmware targets compile.** PlatformIO 6.2.0 (in the project venv) built each target
  successfully:

  | Target | Platform | Result |
  |---|---|---|
  | `firmware/arduino` (`uno_bench`) | `atmelavr@5.1.0` | **SUCCESS** — RAM 356 B (17.4%), flash 2358 B (7.3%) |
  | `firmware/esp32_cam` (`esp32cam`) | `espressif32@6.9.0` | **SUCCESS** — RAM 48364 B (14.8%), flash 839897 B (26.7%) |

  One real compile error was found and fixed while establishing this:

  - `firmware/esp32_cam/src/main.cpp` used `HTTPD_503_SERVICE_UNAVAILABLE`, which **does not
    exist** in the Arduino-ESP32 2.x `httpd_err_code_t` enum. The SDK header
    (`esp_http_server/include/esp_http_server.h`) defines only 400, 401, 403, 404, 405, 408,
    411, 414, 431, 500, 501 and 505. Replaced with `HTTPD_501_METHOD_NOT_IMPLEMENTED`, the
    correct available code for "this operation is not available in this build", which preserves
    the original intent ("UART not configured").

  The ESP32-CAM target requires `src/secrets.h`, copied from `src/config.example.h`. It was built
  with a temporary local copy of the example file containing placeholder values; the copy was
  deleted afterwards and no credential was committed. `secrets.h` is gitignored.

## Not established

- **Browser interaction / visual review.** Not performed. API/static-file checks and JavaScript
  syntax checks do not establish browser QA.
- **Hardware testing.** The firmware compiles, but **no board has been flashed and no hardware
  has been exercised.** A successful build is not a hardware test. See the claim table below.
- **Actual camera, UART, sensors, driver, physical stopping or autonomous movement.** No hardware
  connected.
- **The binary movement protocol.** `docs/PROTOCOL-v2.md` specifies the v2 framing, motion and
  telemetry frames, but **no implementation of it exists in this repository**. Neither board
  implements COBS framing or CRC-8, and the Arduino still parses the newline-ASCII bench protocol
  only. See [PROTOCOL-v2.md](PROTOCOL-v2.md) §7.
- **The end-to-end freshness rule.** ADR-0004 specifies the freshness model (session id,
  monotonic sequence, timestamp age, bounded uncertainty). It is **not implemented**. No code in
  this repository accepts, ages or rejects a motion command.
- **Physical timing and stopping distance.** Not measured. The 300 ms watchdog and the
  20 cm / 15 cm emergency thresholds in the v2 documents are **provisional design values**, not
  measured limits — see [adr/ADR-0006.md](adr/ADR-0006.md).

## Reproduce locally

```powershell
mkdir -p runtime/pytest-tmp
.\.venv\Scripts\python.exe -m pytest -q --basetemp=runtime/pytest-tmp
.\.venv\Scripts\ruff.exe check .
node --check taxi/static/app.js
```

`ruff` and `python` are not on `PATH` in this environment — use the project venv, or the `py`
launcher.

For board builds, follow [HARDWARE.md](HARDWARE.md) and record compiler logs and exact board
settings. For browser QA, run the demo steps in README.md at desktop and phone widths.

## Recording hardware claims

Three separate claims, never merged:

| Claim | What it means |
|---|---|
| Target build | The firmware compiled for a target. |
| Hardware test | The board was flashed, connected, and observed behaving as expected. |
| Motion validation | The vehicle moved under control, with stopping measured. |

A successful compile is not a hardware test. Simulation is not physical validation.
