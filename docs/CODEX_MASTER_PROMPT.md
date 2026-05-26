# Codex Master Prompt

Use this prompt to start the project in Codex.

```text
You are working on an industrial IoT platform for ESP32 gateways and PLCs.

Core architecture:
- PLC communicates with ESP32 over Modbus TCP/IP.
- PLC is the Modbus TCP Server.
- ESP32 is the Modbus TCP Client.
- ESP32 sends telemetry, health, connection events, config ACKs, and command ACKs to the IoT Server over MQTT.
- Backend provides REST APIs and WebSocket realtime updates.
- Dashboard manages provisioning, device config, PLC config, signal mapping, realtime data, history charts, alerts, reports, and users.

Important product requirements:
- Add new ESP32 devices easily through ESP32 WiFi AP setup portal.
- Server generates one-time claim tokens and QR payloads.
- ESP32 sends MAC-based hardware_id and mac_address during claim.
- Server must enforce unique hardware_id and unique mac_address.
- MAC is used for hardware identity only; do not use raw MAC as main display name.
- Backend must issue per-device MQTT credentials after claim.
- Support WiFi RSSI/quality monitoring.
- Detect and report PLC connection errors.
- Detect and report MQTT/server connection errors.
- Implement reconnect logic and R_CONNECT_SUCCESS / R_CONNECT_FAILED events.
- Store telemetry history and draw realtime/historical charts.
- Support CSV/XLSX export.
- Support admin/operator/viewer permissions.

Please read AGENTS.md, PROJECT_MAP.md, IMPLEMENTATION_PLAN.md, and API_AND_MQTT_CONTRACT.md first. Then implement the next smallest useful increment. Prefer small, tested commits.

Start with Phase 1 if the repository is empty:
1. Create backend skeleton with FastAPI.
2. Add database models and migrations for devices, telemetry, and health.
3. Add MQTT topic constants and ingestion handlers.
4. Add a simple WebSocket broadcaster.
5. Add minimal README and local Docker Compose for Postgres and MQTT broker.

When done, report:
- Files created/changed.
- Commands run.
- Tests/build result.
- What should be implemented next.
```

