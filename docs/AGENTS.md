# AGENTS.md - Codex Instructions for ESP32 Modbus TCP IoT Server

## Project Mission

Build an industrial IoT platform where ESP32 gateways communicate with PLCs via Modbus TCP/IP, publish telemetry/health/events to a server via MQTT, and are managed from a web dashboard with provisioning, signal mapping, realtime charts, alerts, reports, and admin/user permissions.

## Non-Negotiable Architecture

- PLC is the Modbus TCP Server.
- ESP32 is the Modbus TCP Client.
- ESP32 communicates with the IoT Server using MQTT for telemetry, health, events, config ACK, and commands.
- Server provides REST APIs for management and WebSocket for realtime dashboard updates.
- Device provisioning uses ESP32 WiFi AP setup portal plus one-time claim token from the server.
- New ESP32 devices must be uniquely identified by MAC-based `hardware_id` and `mac_address`.
- `hardware_id` and `mac_address` must have unique constraints on the server.
- Do not use raw MAC address as the main display name or primary MQTT topic identity.
- Use `device_id` UUID internally and `device_code` for human-friendly UI and MQTT topic paths.

## Build Order

Always implement in small, testable increments:

1. Basic backend project and database models.
2. MQTT ingestion for telemetry and health.
3. Device CRUD and provisioning sessions.
4. ESP32 firmware setup portal and claim flow.
5. Modbus TCP polling for one register.
6. Signal mapping and multi-signal telemetry.
7. Health monitoring and reconnect logic.
8. Alerts and limits.
9. Output command workflow.
10. Reports and export.
11. Production Docker and security hardening.

## Backend Rules

- Prefer FastAPI, SQLAlchemy, Alembic, Pydantic, and PostgreSQL.
- Use clear module boundaries: auth, users, devices, provisioning, plc_connections, signals, telemetry, health, alerts, reports, audit_logs, websocket, mqtt.
- All API input must be validated with Pydantic schemas.
- Store secrets only as hashes where possible.
- Claim tokens must be stored as hashes, one-time use, and expire.
- MQTT passwords must never be returned after initial creation unless rotated.
- Use UTC timestamps in the database.
- Use idempotent handling for MQTT messages when message IDs exist.
- Never create duplicate active alerts for the same device/signal/condition.
- Always create audit logs for config changes, permission changes, and PLC output writes.
- Include tests for critical business logic: provisioning, duplicate MAC rejection, limits, alert lifecycle, permission checks.

## Firmware Rules

- Firmware must support a factory/setup mode and a run mode.
- In factory/setup mode, ESP32 starts a WiFi AP named `IOT_SETUP_{MAC_SUFFIX}`.
- Setup portal must show MAC, hardware ID, firmware version, and setup status.
- Firmware must generate a stable `hardware_id` from MAC/chip ID.
- Firmware must not hard-code per-device server credentials.
- Firmware stores config in NVS/flash only after successful validation.
- Firmware must reconnect network, PLC, and MQTT independently.
- Use backoff for reconnect attempts: 5s, 10s, 30s, then 60s max.
- Publish health periodically.
- Buffer important telemetry/events while MQTT is unavailable, within memory/storage limits.
- Validate all server commands before writing to PLC.
- Only write signals configured as writable.
- Always publish ACK for commands and config updates.

## Frontend Rules

- Prefer React or Next.js with TypeScript.
- Use typed API clients and shared DTO types when available.
- Required pages: Login, Dashboard, Device List, Device Detail, Add Device Wizard, PLC Config, Signal Mapping, Health & Connection, Alerts, History Chart, Reports, User Management.
- Device health must clearly show WiFi RSSI, WiFi quality, PLC status, server/MQTT status, uptime, heap, reconnect counts, and last error.
- Use role-based UI visibility but never rely on frontend-only permission checks.
- Show dangerous write actions with confirmation.
- Show config publish and device ACK status clearly.

## Data Model Rules

Minimum required tables:

- `users`
- `devices`
- `device_hardware`
- `device_credentials`
- `device_provisioning_sessions`
- `plc_connections`
- `signals`
- `signal_limits`
- `telemetry_history`
- `device_health_latest`
- `device_connection_events`
- `alerts`
- `audit_logs`

Use migrations for all schema changes.

## MQTT Topic Rules

Use this topic namespace:

```text
iot/device/{device_code}/telemetry
iot/device/{device_code}/health
iot/device/{device_code}/event
iot/device/{device_code}/status
iot/device/{device_code}/config/set
iot/device/{device_code}/config/ack
iot/device/{device_code}/command
iot/device/{device_code}/command/ack
iot/device/{device_code}/diagnostic/ping
iot/device/{device_code}/diagnostic/result
```

MQTT payloads must include:

- `device_id` or `device_code`
- `timestamp`
- message type specific data
- firmware/config version when relevant
- request ID for commands and ACKs

## Modbus Rules

- Use zero-based Modbus addresses in firmware and backend storage.
- UI may display both user-friendly documented address and zero-based address.
- Document this mapping clearly: PLC document `40001` equals firmware address `0` for holding registers.
- Support these areas: coil, discrete_input, input_register, holding_register.
- Support these data types initially: bool, int16, uint16, int32, uint32, float32.
- Support scale and offset on analog values.
- Support byte order and word order for 32-bit values.
- Do not poll disabled signals.
- Group reads where possible to reduce PLC load.

## Security Rules

- One-time claim token expires after 15-30 minutes.
- Reject claim if MAC/hardware ID is already active.
- Provide admin-only flow for replacing hardware.
- Use separate MQTT credentials per device.
- Do not log raw passwords or tokens.
- Protect output writes with backend permission checks.
- Track all output writes in audit logs.
- Use HTTPS in production.
- Consider MQTT TLS in production.

## Testing Rules

Before considering a task done, run the relevant tests or explain why they cannot run.

Backend:

- Run unit tests.
- Run migration check.
- Run lint/format if configured.

Frontend:

- Run typecheck.
- Run lint.
- Run tests if configured.

Firmware:

- Run compile/build.
- Add mockable classes for Modbus, MQTT, and network logic where possible.

## Documentation Rules

Update docs when changing:

- API contracts.
- MQTT payloads.
- Database schema.
- Provisioning flow.
- Firmware configuration.
- Deployment steps.

## Output Style for Codex Work

When completing a task, summarize:

1. What changed.
2. Files changed.
3. Tests/builds run.
4. Any known limitations or follow-up tasks.

