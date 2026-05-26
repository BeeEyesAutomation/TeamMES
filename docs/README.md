# ESP32 Modbus TCP IoT Server - Codex Starter Pack

This pack contains English planning and rule files for using Codex to implement an ESP32 + PLC Modbus TCP/IP + IoT Server system.

## Files

- `PROJECT_MAP.md` - Target repository structure and module responsibilities.
- `IMPLEMENTATION_PLAN.md` - Phased build plan from MVP to production.
- `AGENTS.md` - Persistent rules and coding instructions for Codex.
- `CODEX_MASTER_PROMPT.md` - A ready-to-paste prompt for starting Codex work.
- `API_AND_MQTT_CONTRACT.md` - Initial API, MQTT topics, payloads, and database contracts.

## Core Architecture

PLC communicates with ESP32 over Modbus TCP/IP. ESP32 communicates with the IoT Server over MQTT. The backend stores telemetry, health, alerts, config, users, and reports. The web dashboard manages device provisioning, signal mapping, realtime status, charts, alarms, exports, and permissions.
