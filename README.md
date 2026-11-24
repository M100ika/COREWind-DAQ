# COREWind DAQ

Information and data acquisition system for gearbox efficiency testing.

## System

**Sensors:** 2 × ZJ-2000AF torque/speed sensors (four pulse channels total), 2 × DS18B20 thermometers; 2 extra pulse channels reserved.

**Acquisition:** ESP32-S3 N16R8 captures raw pulse counts/frequencies and temperature. Signal conditioning is provisionally based on 2 × LM393 dual comparators (not galvanically isolated). Sensor analog/pulse electrical characteristics and calibration must be verified before operation.

**Network and storage:** Raspberry Pi provides an offline Wi-Fi access point, measurement receiver/server, web GUI, and persistent CSV archive on the **Pi microSD**. ESP32 joins Pi's Wi-Fi as a client. No ESP32 microSD module is used in this phase.

**User interface:** Live measurements, experiments (4 gearboxes × 9 regimes), sensor calibration, diagnostics, and archival downloads.

## Instructions for coding agents

**Read [AGENTS.md](AGENTS.md) before making changes.** It is the shared engineering specification for Codex and other AI agents, including required pin assignments, development tasks, verification expectations and safety limitations.

## Planned repository layout

```text
firmware/           ESP32-S3 PlatformIO firmware
server/             Raspberry Pi Python backend
web/                Local, responsive web GUI
docs/               Hardware, protocol, calibration, deployment
tests/              Simulator and automated tests
.github/workflows/  Continuous integration
```

## Status

Specification initialized. Firmware, server, GUI and hardware tests are **not yet implemented or verified**. Do not interpret illustrations or example readings as validated measurements.

Project URL: https://github.com/M100ika/COREWind-DAQ
