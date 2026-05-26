# Project Map: ESP32 Modbus TCP IoT Server

## 1. System Architecture

```text
PLC / Machine
  - Modbus TCP Server
  - IP address, port 502, unit ID
        ⇅ Modbus TCP/IP
ESP32 Gateway
  - Modbus TCP Client
  - WiFi AP provisioning
  - MQTT telemetry client
  - health monitor and reconnect logic
        ⇅ MQTT / HTTP claim API
IoT Server
  - MQTT broker integration
  - REST API
  - WebSocket realtime gateway
  - PostgreSQL / TimescaleDB
        ⇅ REST / WebSocket
Web Dashboard
  - Admin, operator, viewer UI
```

## 2. Recommended Repository Structure

```text
iot-modbus-platform/
├── AGENTS.md
├── README.md
├── docker-compose.yml
├── docs/
│   ├── PROJECT_MAP.md
│   ├── IMPLEMENTATION_PLAN.md
│   ├── API_AND_MQTT_CONTRACT.md
│   ├── SECURITY_MODEL.md
│   └── DEVICE_PROVISIONING.md
│
├── backend/
│   ├── README.md
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── app/
│   │   ├── main.py
│   │   ├── core/
│   │   │   ├── config.py
│   │   │   ├── security.py
│   │   │   ├── logging.py
│   │   │   └── errors.py
│   │   ├── db/
│   │   │   ├── session.py
│   │   │   ├── base.py
│   │   │   └── migrations/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── users/
│   │   │   ├── devices/
│   │   │   ├── provisioning/
│   │   │   ├── plc_connections/
│   │   │   ├── signals/
│   │   │   ├── telemetry/
│   │   │   ├── health/
│   │   │   ├── alerts/
│   │   │   ├── reports/
│   │   │   ├── audit_logs/
│   │   │   └── websocket/
│   │   ├── mqtt/
│   │   │   ├── client.py
│   │   │   ├── handlers.py
│   │   │   └── topics.py
│   │   └── tests/
│   └── scripts/
│
├── frontend/
│   ├── README.md
│   ├── package.json
│   ├── src/
│   │   ├── app/
│   │   ├── api/
│   │   ├── components/
│   │   ├── features/
│   │   │   ├── auth/
│   │   │   ├── dashboard/
│   │   │   ├── devices/
│   │   │   ├── provisioning/
│   │   │   ├── plc-config/
│   │   │   ├── signals/
│   │   │   ├── charts/
│   │   │   ├── alerts/
│   │   │   ├── reports/
│   │   │   └── users/
│   │   ├── hooks/
│   │   ├── lib/
│   │   ├── routes/
│   │   └── types/
│   └── tests/
│
├── firmware/
│   ├── README.md
│   ├── platformio.ini
│   ├── src/
│   │   ├── main.cpp
│   │   ├── config/
│   │   │   ├── ConfigManager.h
│   │   │   └── ConfigManager.cpp
│   │   ├── provisioning/
│   │   │   ├── SetupPortal.h
│   │   │   ├── SetupPortal.cpp
│   │   │   ├── ClaimClient.h
│   │   │   └── ClaimClient.cpp
│   │   ├── network/
│   │   │   ├── NetworkManager.h
│   │   │   └── NetworkManager.cpp
│   │   ├── plc/
│   │   │   ├── ModbusTcpClient.h
│   │   │   ├── ModbusTcpClient.cpp
│   │   │   ├── SignalMapper.h
│   │   │   └── SignalMapper.cpp
│   │   ├── mqtt/
│   │   │   ├── MqttClient.h
│   │   │   ├── MqttClient.cpp
│   │   │   └── Topics.h
│   │   ├── health/
│   │   │   ├── HealthMonitor.h
│   │   │   └── HealthMonitor.cpp
│   │   ├── buffer/
│   │   │   ├── LocalBuffer.h
│   │   │   └── LocalBuffer.cpp
│   │   └── commands/
│   │       ├── CommandHandler.h
│   │       └── CommandHandler.cpp
│   └── test/
│
└── infra/
    ├── nginx/
    ├── emqx/
    ├── postgres/
    ├── grafana/
    └── scripts/
```

## 3. Module Responsibilities

### Firmware

- `ConfigManager`: stores device config in NVS/flash.
- `SetupPortal`: creates ESP32 local WiFi AP and setup web portal.
- `ClaimClient`: sends MAC-based hardware ID and claim token to server.
- `NetworkManager`: manages WiFi/Ethernet, RSSI, reconnect, IP settings.
- `ModbusTcpClient`: reads/writes PLC registers/coils over Modbus TCP/IP.
- `SignalMapper`: converts raw Modbus values to engineering values using data type, byte order, scale, offset, and unit.
- `MqttClient`: publishes telemetry, health, events, and receives commands/config.
- `HealthMonitor`: tracks WiFi RSSI, PLC connection, server connection, uptime, heap, reconnect counts.
- `LocalBuffer`: stores telemetry/events while MQTT/server is unavailable and replays after reconnect.
- `CommandHandler`: validates server commands and writes allowed outputs to PLC.

### Backend

- `auth`: login, JWT/session, password hashing.
- `users`: admin/operator/viewer roles and device-level permissions.
- `devices`: CRUD for device metadata and lifecycle.
- `provisioning`: claim-token generation, MAC/hardware ID linking, MQTT credential issuing.
- `plc_connections`: PLC IP, port, unit ID, timeout, poll interval.
- `signals`: Modbus register/coil mapping, scale, offset, limits.
- `telemetry`: ingest MQTT telemetry and store time-series history.
- `health`: latest health snapshot and connection event history.
- `alerts`: limit alarms, connection alarms, acknowledge/resolution workflow.
- `reports`: CSV/XLSX export.
- `audit_logs`: record config changes and output-control actions.
- `websocket`: push realtime telemetry, health, and alerts to dashboard.

### Frontend

- `dashboard`: overview cards, realtime telemetry, online/offline status.
- `devices`: list, create, edit, disable, detail.
- `provisioning`: Add ESP32 wizard with QR/claim token.
- `plc-config`: Modbus TCP configuration.
- `signals`: signal mapping, limits, units, access mode.
- `charts`: realtime and historical charts.
- `alerts`: active and historical alarms.
- `reports`: data export.
- `users`: roles and permissions.

## 4. Naming Standards

- Device hardware identity: `hardware_id`, derived from MAC and chip ID.
- Human-friendly device code: `device_code`, for example `ESP32_LINE_01`.
- Database primary key: `device_id`, UUID.
- MQTT topic identity: use `device_code` or UUID, not raw MAC.
- MAC address must be stored as unique hardware identity, not as the main display name.

