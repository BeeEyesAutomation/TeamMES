# Implementation Plan: ESP32 Modbus TCP IoT Server

## Phase 0 - Technical Decisions

Finalize these decisions before coding:

- PLC protocol: Modbus TCP/IP.
- ESP32 role: Modbus TCP client.
- PLC role: Modbus TCP server.
- Server protocol: MQTT for telemetry/health/events/commands.
- Initial backend stack: FastAPI + PostgreSQL + SQLAlchemy/Alembic.
- MQTT broker: EMQX or Mosquitto.
- Frontend stack: React or Next.js + ECharts.
- Provisioning method: ESP32 WiFi AP setup portal + claim token/QR from server.
- Unique hardware identity: MAC-based `hardware_id` with unique server constraint.

## Phase 1 - MVP Telemetry Pipeline

Goal: read one PLC register and show it in realtime.

### Firmware

- Connect ESP32 to WiFi or Ethernet.
- Connect to PLC over Modbus TCP.
- Read one holding register.
- Convert value using scale/offset.
- Connect to MQTT broker.
- Publish telemetry to `iot/device/{device_code}/telemetry`.
- Publish basic health to `iot/device/{device_code}/health`.

### Backend

- Subscribe to MQTT telemetry.
- Store telemetry in PostgreSQL.
- Store latest device health.
- Expose REST endpoint for latest value.
- Push realtime values over WebSocket.

### Frontend

- Login screen.
- Device list page.
- Device detail page showing latest value and connection status.

### Acceptance Criteria

- One ESP32 reads one register from PLC simulator or real PLC.
- Server receives and stores values.
- Dashboard updates without page refresh.
- Device online/offline status works.

## Phase 2 - Device Provisioning

Goal: add a new ESP32 without custom firmware per device.

### Backend

- Add provisioning session API.
- Generate one-time claim token.
- Generate QR payload.
- Store claim token hash, expiry, and status.
- Add `hardware_id`, `mac_address`, and `chip_id` unique constraints.
- Issue per-device MQTT credentials after claim.

### Firmware

- If no config exists, start WiFi AP: `IOT_SETUP_{MAC_SUFFIX}`.
- Host setup portal at `192.168.4.1`.
- Show hardware ID, MAC, firmware version.
- Accept server host, MQTT host, network config, PLC config, and claim token.
- Send claim request to server.
- Save returned device code and MQTT credential.
- Reboot into run mode.

### Frontend

- Add Device wizard.
- Show claim token and QR code.
- Show waiting/claimed/provisioned states.
- Show detected MAC/hardware ID after claim.

### Acceptance Criteria

- A factory-reset ESP32 can be linked to server using a claim token.
- Duplicate MAC/hardware ID is rejected.
- Claim token is one-time and expires.

## Phase 3 - Signal Mapping and PLC Config

Goal: configure PLC registers/coils from the dashboard.

### Backend

- CRUD for PLC connection: IP, port, unit ID, timeout, poll interval.
- CRUD for signal mappings.
- Support signal types: analog, digital_in, digital_out.
- Support Modbus areas: coil, discrete_input, input_register, holding_register.
- Support data types: bool, int16, uint16, int32, uint32, float32.
- Support word order and byte order.
- Publish config updates to ESP32 over MQTT.
- Track config version and config ACK.

### Firmware

- Receive config update.
- Validate config.
- Apply config without reboot where possible.
- ACK success/failure with error details.
- Poll multiple signals.

### Frontend

- PLC config form.
- Signal mapping table.
- Limit configuration UI.
- Config publish status and ACK display.

### Acceptance Criteria

- Admin can define several Modbus signals from dashboard.
- ESP32 applies config and reports ACK.
- Telemetry includes all enabled signals.

## Phase 4 - Health Monitoring and Reconnect

Goal: detect WiFi, PLC, and server connection problems.

### Firmware

- Measure WiFi RSSI and quality percentage.
- Track network reconnect count.
- Track PLC connection status, latency, fail count, last error.
- Track MQTT/server status, last publish, last ACK, reconnect count.
- Implement reconnect backoff: 5s, 10s, 30s, 60s.
- Publish health periodically.
- Publish connection events.
- Buffer events/telemetry while MQTT is unavailable.
- Replay buffered data after reconnect.

### Backend

- Store latest health in `device_health_latest`.
- Store connection events in `device_connection_events`.
- Create alerts for weak WiFi, PLC failure, MQTT/server disconnection, and device offline.
- Resolve alerts on reconnect.

### Frontend

- Add Health & Connection tab.
- Show WiFi RSSI/quality, PLC status, server status, uptime, heap, reconnect counts.
- Show connection event history.

### Acceptance Criteria

- Pulling PLC network cable creates PLC connection alarm.
- Stopping MQTT broker creates server connection alarm after reconnect.
- Weak WiFi threshold creates warning/alarm.
- Reconnect success creates resolved event.

## Phase 5 - Alerts and Limits

Goal: provide industrial alarm workflow.

### Backend

- Evaluate warning/alarm/critical limits.
- Support hysteresis.
- Avoid duplicate active alerts for the same device/signal/condition.
- Support acknowledge and resolve workflow.
- Push alert events through WebSocket.
- Store audit logs for acknowledgement.

### Frontend

- Active alert list.
- Historical alert list.
- Acknowledge button with permission check.
- Alert filters by device, signal, level, time.

### Acceptance Criteria

- Limit crossing creates alert.
- Returning to normal resolves alert according to hysteresis rule.
- User acknowledgement is stored.

## Phase 6 - Output Control

Goal: allow safe PLC coil/register writes from dashboard.

### Backend

- Check user permission before write command.
- Only allow writes to signals with `access=write` or `read_write`.
- Publish command with request ID.
- Wait for ESP32 ACK.
- Store audit log with before/after values.

### Firmware

- Receive command.
- Validate signal exists and is writable.
- Write Modbus coil/register.
- Publish command ACK success/failure.

### Frontend

- Control widgets for writable signals.
- Confirmation modal for critical writes.
- Show command result and audit history.

### Acceptance Criteria

- Unauthorized user cannot write output.
- Authorized command writes PLC and receives ACK.
- Failed Modbus write is shown clearly.

## Phase 7 - Reports and Export

Goal: export history and alerts for operations.

### Backend

- Query telemetry by device, signal, time range.
- Support raw, minute average, hourly average, and daily summary.
- Export CSV.
- Export XLSX with sheets: Raw Data, Summary, Alerts.

### Frontend

- Report page with filters.
- Download CSV/XLSX.

### Acceptance Criteria

- User can export selected signals for selected time range.
- Export contains timestamp, device, signal, value, unit, status.

## Phase 8 - Production Hardening

- Docker Compose for backend, frontend, MQTT broker, PostgreSQL, Nginx.
- HTTPS termination.
- MQTT username/password per device.
- Optional MQTT TLS.
- Database backup script.
- Structured logs.
- Rate limits for claim API.
- Health endpoints for services.
- Deployment documentation.

