# AGENTS.md — COREWind DAQ

## Purpose and working rules

You are a software/electronics engineering agent working on COREWind DAQ, a real measurement and logging system for a gearbox-efficiency test bench.

Repository: https://github.com/M100ika/COREWind-DAQ
Project: COREWind — gearbox torque, speed, temperature and efficiency monitoring.

IMPORTANT:
- Do the work, not just write a plan. Inspect the repository first and preserve existing working code.
- Work on a feature branch and submit a pull request when connected to GitHub; do not overwrite main or force-push.
- Prefer incremental, compilable/testable steps. Never claim hardware has been tested without real measurements.
- Distinguish VERIFIED facts, assumptions, and hardware measurements still needed.
- Communicate with the owner in Russian, concisely. Code, symbols and technical comments can be in English.
- All critical configuration (Wi-Fi SSID, credentials, sensor factors, host address and pin mapping) should be documented/configurable. Never commit passwords, tokens or secrets.
- The system must start after power loss, report health, and avoid silently fabricating or replacing missing measurements.

## Confirmed system scope (current iteration)

Hardware:
- One ESP32-S3 N16R8 development board, programmed using PlatformIO / ESP-IDF-compatible approach (prefer PlatformIO with Arduino framework for MVP, justify deviation).
- Two ZJ-2000AF torque + rotation-speed sensors. Each has Torque (frequency) and Speed (pulse) outputs.
- Two DS18B20 digital temperature probes on one 1-Wire bus.
- Two LM393 dual comparators (4 inputs total) for conditioning frequency/pulse outputs, with output pull-ups to 3.3V. No galvanic isolation is supplied by LM393.
- Two additional GPIO pulse channels reserved for optional inductive speed sensors; no sensors installed now.
- Raspberry Pi (specific model/OS to be discovered) runs Wi-Fi Access Point, web server, API, local storage and GUI. Raspberry Pi microSD is storage. NO microSD module on ESP32 in this phase.
- Sensor power is nominally +15V/0/-15V from two isolated 15V bench supplies, subject to sensor manual verification. ESP32 powered from a separate regulated 5V supply, Pi from its own suitable supply.
- The system must remain usable without an Internet connection.

GPIO assignments (fixed until hardware verification):
| GPIO | Purpose |
| ---- | ------- |
| 4 | ZJ1 Torque frequency |
| 5 | ZJ1 Speed pulses |
| 6 | ZJ2 Torque frequency |
| 7 | ZJ2 Speed pulses |
| 8 | Spare external speed pulse input 1 |
| 9 | Spare external speed pulse input 2 |
| 10 | 1-Wire, both DS18B20 |
| 11,12,13,14 | Reserved SPI/microSD for future ESP32 expansion; leave unused |
| 15,16 | Reserved I2C |
| 17 | Optional local start/stop button |
| 18 | Optional status LED |
| 21 | Spare |

Before wiring, check actual ESP32-S3 board variant, pin availability, boot/USB interactions and GPIO input electrical limits. Never connect ~12V sensor output directly to ESP32. Confirm signal amplitude, polarity, sensor output source impedance and thresholds using oscilloscope, especially Torque ~20.26kHz at rest and Speed signal during shaft rotation.

Current proposed LM393 hardware values (NOT verified or finalized):
- Signal input divider nominal 47k/10k, VCC comparator 5V, output open collector with 4.7k pull-up to ESP32 3.3V.
- Comparator reference nominal 39k/10k from 5V, ~1.02V.
- Hysteresis feedback nominal 470k; exact topology and threshold calculations must be verified by the hardware engineer BEFORE assembly.
- Protection components (including whether 1N5819 is appropriate) are provisional; never assume interchangeable signal-clamp diode parameters.
- Common grounds are required for this non-isolated approach; evaluate grounding and EMI risk near VFD, and recommend isolation upgrade for actual stand deployment.
- The 4 counter channels should be captured with hardware pulse-counter units (PCNT) or an equivalent proven method, accounting for ~20kHz signals, overflow and no missed counts; reserve 2 low-rate counters for later.

## Architecture

### ESP32 firmware
- Acquire 4 frequency/pulse channels continuously; use time-gated measurements with accurate measured elapsed time, not naive timing assumptions.
- Read two DS18B20 by stable unique ROM addresses, with configurable mapping to physical probes.
- Keep acquisition isolated from network task (queues/ring buffer/FreeRTOS appropriate). Avoid blocking 1-Wire conversion or Wi-Fi code in timing-critical code.
- Every record has monotonic sequence ID, uptime/monotonic timestamp in microseconds or milliseconds, actual acquisition window duration, raw counters and derived frequencies, temperature values and sensor-validity/status flags.
- Implement Wi-Fi STA connection to Raspberry Pi AP, automatic reconnect with capped exponential backoff, heartbeat/status, and bounded RAM backlog. On reconnect send queued records oldest-first; detect/report buffer loss explicitly, never silently drop data.
- Avoid depending on ESP32 wall-clock for archival timestamps; Raspberry Pi maintains real time, but preserve ESP monotonic times and session boot ID for correct ordering and gap detection.
- Support offline firmware tests using synthetic sensor data or simulator/mocked counter source.
- Only implement commands that are needed, with clear safety/authorization checks. Sampling is autonomous; start/stop means experiment session recording on Pi, not stopping raw sensor acquisition.

### Raspberry Pi
- Create a Wi-Fi AP with configurable SSID and passphrase. Document setup for the installed Raspberry Pi OS / Wi-Fi adapter, with hostapd/dnsmasq or NetworkManager as applicable; detect conflicts before applying network changes. Do not hardcode interface names.
- Runs Python 3 web service (suggest FastAPI + Uvicorn; optional WebSocket) accessible to PC/phone connected to Pi AP. Use host binding 0.0.0.0 behind local AP, but implement basic authentication or protected controls and avoid unauthenticated control endpoints.
- Receive ESP32 measurements via documented, versioned transport (choose one primary: ESP -> Pi HTTP POST batches or persistent TCP/WebSocket with acknowledgements). Prioritize reliable delivery and implement ACK of highest persisted sequence per ESP boot ID, retries and idempotent deduplication.
- Persist raw measurements immediately to durable local storage on Pi microSD. CSV export is mandatory. SQLite may be used internally for robustness, but deliver one-click CSV download, stable schemas, and recoverable sessions. Flush/sync policy should balance storage endurance and loss risk.
- Experiment metadata includes gearbox ID (4 gearbox options), run regime (1..9), run ID, operator, date/time, sampling interval, calibration version and annotations.
- Server receives data continuously; recording is gated by experiment session state. Auto-start and orderly shutdown should be provided via systemd service.
- If network disappears, UI clearly shows stale/disconnected and data gaps; never invent values. When recovered, reconcile delayed data without duplicates.
- Provide /health, acquisition status, API documentation, log rotation, disk-space alerts, and clear permission/data-directory handling.

### Web GUI
- Responsive, readable, Russian-first; no external CDN or mandatory Internet.
- Dashboard: live frequency, torque, RPM for both sensors, two temperatures, input/output powers, efficiency, recording status, connectivity, graph plots and last update.
- Experiment: gearbox/run selection (4x9), start/stop session, operator notes.
- Calibration: independently calibrate ZJ1 and ZJ2; save coefficients with version, date, input samples and traceable metadata. Do not accept arbitrary default frequency-to-torque conversion as a valid calibration.
- Archive: list/download CSV sessions, export raw and processed values.
- Diagnostics: pulse counts, observed frequency, data age, sensor missing/error, network reconnects, dropped samples, Wi-Fi RSSI, uptime, recording/backlog/disk status.
- UI should work locally offline and degrade gracefully under connection loss.

## Measurement and math

- ZJ Speed output: assumed 60 pulses per revolution pending datasheet/bench verification. RPM = 60 * f_Hz / pulses_per_rev (with PPR configurable); for 60 PPR RPM numerically equals Hz.
- Optional external inductive sensor: assumed 1 pulse/revolution only when installed and verified; for low RPM, implement period measurement rather than 1-second event counting alone.
- Torque is represented by frequency, but exact transfer function (offset, scaling, positive/negative direction) is UNKNOWN. Initial measured static frequency near 20.26kHz on one sensor is an observation, not a verified universal zero or calibration. Store raw frequency. Mark torque, power and efficiency UNCALIBRATED/NULL until calibrated.
- Calibration can start with linear model M_Nm = K * (f_Hz - f0_Hz) only if experiment confirms appropriateness; define per-sensor coefficient set, directions and validity range. Support load-arm mass/length calculation M = m * 9.80665 * L (perpendicular force), using safe fixture and test procedure.
- Mechanical power: P_W = 2*pi*(RPM/60)*Torque_Nm. Document shaft direction and sign convention. Efficiency = 100 * P_out / P_in for physically valid steady-state conditions only; mark invalid/unavailable when P_in~0, sensor error, incomplete calibration or ambiguous sign.
- Distinguish actual zero reading from missing signal; use null/status rather than silently writing zero.
- Capture samples once per approximately 1 second using actual sampling interval and document the jitter.

## Data schema (proposed; finalize & version)
{
  "schema_version": 1,
  "device_id": "esp32-s3-01",
  "boot_id": "boot-unique",
  "seq": 1234,
  "monotonic_ms": 1234567,
  "window_ms": 1000.0,
  "counts": { "torque_1": 20260, "speed_1": 1500, "torque_2": 20250, "speed_2": 300 },
  "frequency_hz": { "torque_1": 20260.0, "speed_1": 1500.0, "torque_2": 20250.0, "speed_2": 300.0 },
  "temperature_c": { "temp_1": 42.6, "temp_2": 38.2 },
  "status": { "torque_1": "OK", "speed_1": "OK", "torque_2": "OK", "speed_2": "OK", "temp_1": "OK", "temp_2": "OK" }
}
This is illustrative data only; do not hard-code sample readings as actual measurements. Define JSON/CSV schemas and explicit units.

## Deliverable structure (adapt if repository already has code)
- AGENTS.md (these instructions)
- README.md (quick start and system overview)
- firmware/ (PlatformIO ESP32-S3 project; pin config, pulse counters, OneWire, Wi-Fi, transmission)
- server/ (Python web service, ingestion, archive/database, API, calibration and tests)
- web/ (local web GUI; avoid network dependencies)
- docs/hardware.md (wiring, power, BOM, test points, risks)
- docs/protocol.md (versioned message, reliability, timestamps and errors)
- docs/calibration.md (procedure and calculation)
- docs/deployment.md (Pi AP, service installation, troubleshooting)
- tests/ (simulation, API, persistence, duplicate/reconnect tests)
- .github/workflows/ (CI for firmware build and Python/test checks)
- .gitignore, config examples, environment sample (no secrets)

## Implementation order

1. Inspect repository; report actual contents. Establish clean skeleton, README and agent instructions.
2. Implement end-to-end mock/simulator: fake ESP sender -> Pi receiver -> persistent storage -> live GUI -> CSV download. Write automated tests.
3. Implement ESP firmware 4 hardware pulse channels, DS18B20, Wi-Fi STA, record queue, automatic reconnect and ACK protocol. Make it compile in CI.
4. Implement Pi Wi-Fi AP installation/configuration instructions and systemd service, without disrupting a live host unless operator explicitly runs setup.
5. Calibration UI/logic, experiment session metadata, power and efficiency math with strict validity gating.
6. Verify diagnostics, data gaps, deduplication, timestamping and restart/recovery behavior.
7. Prepare physical wiring and bench-validation checklist; request or report missing oscilloscope measurements and sensor-manual specifics before declaring hardware verified.

## Acceptance criteria

- Both ESP firmware and Pi service start from documented commands; CI compiles firmware and tests server.
- Mock mode exercises entire flow with no electronics attached.
- 4 working pulse channels are mapped to GPIO4–7; two unused reserved on GPIO8–9; DS18B20 on GPIO10.
- Pi AP serves local GUI with no Internet; ESP joins as client. Pi stores data on its own microSD, ESP does NOT attempt SD logging.
- One record per nominal 1s interval with actual window measurement; no silent packet duplication, counter overflow or loss.
- Data and calibration survive process and power restarts; data gaps or unsaved records are reported.
- Four gearboxes, nine regimes each, session CSV downloads and independent ZJ calibrations are supported.
- Uncalibrated values are never presented as valid torque/power/efficiency.
- Docs clearly label unverified hardware claims and tests, especially LM393 conditioning, ZJ Speed output, power wiring and isolation risk.
- Final agent response contains list of changed files, commands run, test output, limitations and pull-request URL.

## Safety and integrity
- Physical test stand is rotating machinery: no autonomous actuation of the VFD/motor, no automatic commands to torque/speed actuators, and no claim to make the stand safe.
- Do not connect live mains or reconfigure a powered test stand automatically. Sensor wiring and grounding must be validated by an authorized person.
- Do not erase existing Pi SD-card content, change network/AP configuration on a remote Pi, overwrite production data or merge to main without explicit instruction.
- Never fake successful hardware verification, external documentation access or benchmark results.
