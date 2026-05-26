# API and MQTT Contract

## 1. REST API Draft

### Auth

```http
POST /api/auth/login
POST /api/auth/logout
GET  /api/auth/me
```

### Users

```http
GET    /api/users
POST   /api/users
GET    /api/users/{user_id}
PATCH  /api/users/{user_id}
DELETE /api/users/{user_id}
```

### Devices

```http
GET    /api/devices
POST   /api/devices
GET    /api/devices/{device_id}
PATCH  /api/devices/{device_id}
DELETE /api/devices/{device_id}
POST   /api/devices/{device_id}/disable
POST   /api/devices/{device_id}/replace-hardware
```

### Provisioning

```http
POST /api/provisioning/sessions
GET  /api/provisioning/sessions/{session_id}
POST /api/device/claim
```

#### Create Provisioning Session

Request:

```json
{
  "device_code": "ESP32_LINE_01",
  "device_name": "Line 1 Machine Gateway",
  "location": "Factory A / Line 1",
  "expires_in_minutes": 30
}
```

Response:

```json
{
  "session_id": "uuid",
  "device_id": "uuid",
  "device_code": "ESP32_LINE_01",
  "claim_token": "8F2K-91PQ-X7LA",
  "qr_payload": "iot://claim?server=192.168.1.100&token=8F2K-91PQ-X7LA",
  "expires_at": "2026-05-26T10:30:00Z"
}
```

#### ESP32 Claim Device

Request:

```json
{
  "claim_token": "8F2K-91PQ-X7LA",
  "hardware_id": "ESP32-A4CF12345678",
  "mac": "A4:CF:12:34:56:78",
  "chip_id": "ABC123456",
  "firmware_version": "1.0.0",
  "setup_method": "wifi_ap"
}
```

Response:

```json
{
  "status": "success",
  "device_id": "uuid",
  "device_code": "ESP32_LINE_01",
  "mqtt": {
    "host": "192.168.1.100",
    "port": 1883,
    "username": "ESP32_LINE_01",
    "password": "generated_password"
  },
  "config_version": 1
}
```

### PLC Connection

```http
GET   /api/devices/{device_id}/plc-connection
PUT   /api/devices/{device_id}/plc-connection
POST  /api/devices/{device_id}/plc-connection/test
```

PLC config body:

```json
{
  "protocol": "modbus_tcp",
  "host": "192.168.1.10",
  "port": 502,
  "unit_id": 1,
  "timeout_ms": 1000,
  "poll_interval_ms": 1000,
  "enabled": true
}
```

### Signals

```http
GET    /api/devices/{device_id}/signals
POST   /api/devices/{device_id}/signals
GET    /api/signals/{signal_id}
PATCH  /api/signals/{signal_id}
DELETE /api/signals/{signal_id}
```

Signal body:

```json
{
  "key": "temperature",
  "name": "Temperature",
  "signal_type": "analog",
  "access_mode": "read",
  "modbus_area": "holding_register",
  "address": 0,
  "data_type": "int16",
  "word_order": "abcd",
  "byte_order": "big_endian",
  "scale": 0.1,
  "offset": 0,
  "unit": "degC",
  "enabled": true,
  "limits": {
    "warning_high": 70,
    "alarm_high": 80,
    "critical_high": 90,
    "hysteresis": 2
  }
}
```

### Telemetry and History

```http
GET /api/devices/{device_id}/telemetry/latest
GET /api/telemetry/history?device_id=...&signal_id=...&from=...&to=...
```

### Health

```http
GET /api/devices/{device_id}/health/latest
GET /api/devices/{device_id}/connection-events
```

### Alerts

```http
GET  /api/alerts
POST /api/alerts/{alert_id}/acknowledge
POST /api/alerts/{alert_id}/resolve
```

### Reports

```http
GET /api/reports/telemetry.csv?device_id=...&from=...&to=...
GET /api/reports/telemetry.xlsx?device_id=...&from=...&to=...
```

## 2. MQTT Topics

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

## 3. MQTT Payloads

### Telemetry

```json
{
  "device_code": "ESP32_LINE_01",
  "timestamp": "2026-05-26T10:00:00Z",
  "config_version": 12,
  "signals": {
    "temperature": 72.4,
    "pressure": 3.21,
    "motor_running": true,
    "pump_start": false
  },
  "quality": "good"
}
```

### Health

```json
{
  "device_code": "ESP32_LINE_01",
  "timestamp": "2026-05-26T10:00:00Z",
  "network": {
    "mode": "wifi",
    "ip": "192.168.1.50",
    "wifi_rssi": -61,
    "wifi_quality": 78,
    "wifi_ssid": "Factory_WiFi",
    "mac": "A4:CF:12:34:56:78",
    "network_reconnect_count": 2
  },
  "connections": {
    "plc": {
      "status": "connected",
      "latency_ms": 12,
      "last_success_at": "2026-05-26T09:59:59Z",
      "last_error_at": null,
      "error_count": 0,
      "last_error": null,
      "plc_reconnect_count": 1
    },
    "server": {
      "mqtt_status": "connected",
      "last_publish_at": "2026-05-26T10:00:00Z",
      "last_ack_at": "2026-05-26T10:00:00Z",
      "error_count": 0,
      "last_error": null,
      "mqtt_reconnect_count": 1
    }
  },
  "runtime": {
    "uptime_sec": 86400,
    "free_heap": 153000,
    "firmware_version": "1.0.0",
    "config_version": 12
  }
}
```

### Connection Event

```json
{
  "device_code": "ESP32_LINE_01",
  "event_type": "PLC_CONNECT_ERROR",
  "target": "PLC",
  "level": "alarm",
  "timestamp": "2026-05-26T10:00:00Z",
  "message": "Cannot connect PLC 192.168.1.10:502",
  "detail": {
    "plc_ip": "192.168.1.10",
    "port": 502,
    "timeout_ms": 1000,
    "fail_count": 5,
    "last_error": "TCP_CONNECT_TIMEOUT"
  }
}
```

### Reconnect Success Event

```json
{
  "device_code": "ESP32_LINE_01",
  "event_type": "R_CONNECT_SUCCESS",
  "target": "PLC",
  "level": "info",
  "timestamp": "2026-05-26T10:02:00Z",
  "message": "PLC reconnected successfully",
  "detail": {
    "downtime_sec": 120,
    "attempts": 4
  }
}
```

### Config Set

```json
{
  "request_id": "cfg_20260526_001",
  "device_code": "ESP32_LINE_01",
  "config_version": 13,
  "plc": {
    "protocol": "modbus_tcp",
    "host": "192.168.1.10",
    "port": 502,
    "unit_id": 1,
    "timeout_ms": 1000,
    "poll_interval_ms": 1000
  },
  "signals": [
    {
      "key": "temperature",
      "name": "Temperature",
      "signal_type": "analog",
      "access_mode": "read",
      "modbus_area": "holding_register",
      "address": 0,
      "data_type": "int16",
      "scale": 0.1,
      "offset": 0,
      "unit": "degC",
      "enabled": true
    }
  ]
}
```

### Config ACK

```json
{
  "request_id": "cfg_20260526_001",
  "device_code": "ESP32_LINE_01",
  "status": "success",
  "config_version": 13,
  "timestamp": "2026-05-26T10:00:02Z",
  "error": null
}
```

### Command

```json
{
  "request_id": "cmd_20260526_001",
  "device_code": "ESP32_LINE_01",
  "signal_key": "pump_start",
  "value": true,
  "issued_by": "admin@example.com",
  "timestamp": "2026-05-26T10:05:00Z"
}
```

### Command ACK

```json
{
  "request_id": "cmd_20260526_001",
  "device_code": "ESP32_LINE_01",
  "signal_key": "pump_start",
  "status": "success",
  "value": true,
  "timestamp": "2026-05-26T10:05:01Z",
  "error": null
}
```

## 4. Database Schema Draft

```sql
CREATE TABLE devices (
  id UUID PRIMARY KEY,
  device_code VARCHAR(100) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  location VARCHAR(255),
  status VARCHAR(50) DEFAULT 'unclaimed',
  last_seen_at TIMESTAMP,
  config_version INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE device_hardware (
  id UUID PRIMARY KEY,
  device_id UUID REFERENCES devices(id),
  hardware_id VARCHAR(100) UNIQUE NOT NULL,
  mac_address VARCHAR(50) UNIQUE NOT NULL,
  chip_id VARCHAR(100),
  first_seen_at TIMESTAMP,
  last_seen_at TIMESTAMP,
  firmware_version VARCHAR(50),
  is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE device_provisioning_sessions (
  id UUID PRIMARY KEY,
  device_id UUID REFERENCES devices(id),
  claim_token_hash TEXT NOT NULL,
  status VARCHAR(30) DEFAULT 'waiting',
  expires_at TIMESTAMP NOT NULL,
  claimed_at TIMESTAMP,
  claimed_hardware_id VARCHAR(100),
  claimed_mac VARCHAR(50),
  created_by UUID,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE device_credentials (
  id UUID PRIMARY KEY,
  device_id UUID REFERENCES devices(id),
  mqtt_username VARCHAR(100) NOT NULL,
  mqtt_password_hash TEXT NOT NULL,
  active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW(),
  rotated_at TIMESTAMP
);

CREATE TABLE plc_connections (
  id UUID PRIMARY KEY,
  device_id UUID REFERENCES devices(id),
  protocol VARCHAR(50) NOT NULL DEFAULT 'modbus_tcp',
  host VARCHAR(100) NOT NULL,
  port INT NOT NULL DEFAULT 502,
  unit_id INT DEFAULT 1,
  timeout_ms INT DEFAULT 1000,
  poll_interval_ms INT DEFAULT 1000,
  is_enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE signals (
  id UUID PRIMARY KEY,
  device_id UUID REFERENCES devices(id),
  signal_key VARCHAR(100) NOT NULL,
  signal_name VARCHAR(255),
  signal_type VARCHAR(50),
  access_mode VARCHAR(20),
  modbus_area VARCHAR(50),
  address INT NOT NULL,
  data_type VARCHAR(50),
  word_order VARCHAR(20),
  byte_order VARCHAR(20),
  scale DOUBLE PRECISION DEFAULT 1,
  offset DOUBLE PRECISION DEFAULT 0,
  unit VARCHAR(20),
  is_enabled BOOLEAN DEFAULT TRUE,
  UNIQUE(device_id, signal_key)
);

CREATE TABLE signal_limits (
  id UUID PRIMARY KEY,
  signal_id UUID REFERENCES signals(id),
  warning_low DOUBLE PRECISION,
  warning_high DOUBLE PRECISION,
  alarm_low DOUBLE PRECISION,
  alarm_high DOUBLE PRECISION,
  critical_low DOUBLE PRECISION,
  critical_high DOUBLE PRECISION,
  hysteresis DOUBLE PRECISION DEFAULT 0
);

CREATE TABLE telemetry_history (
  time TIMESTAMP NOT NULL,
  device_id UUID REFERENCES devices(id),
  signal_id UUID REFERENCES signals(id),
  value DOUBLE PRECISION,
  value_bool BOOLEAN,
  quality VARCHAR(30),
  PRIMARY KEY (time, device_id, signal_id)
);

CREATE TABLE device_health_latest (
  device_id UUID PRIMARY KEY REFERENCES devices(id),
  network_mode VARCHAR(20),
  ip_address VARCHAR(50),
  wifi_rssi INT,
  wifi_quality INT,
  wifi_ssid VARCHAR(100),
  plc_status VARCHAR(50),
  plc_last_success_at TIMESTAMP,
  plc_last_error_at TIMESTAMP,
  plc_error_count INT,
  plc_last_error TEXT,
  server_status VARCHAR(50),
  server_last_publish_at TIMESTAMP,
  server_last_ack_at TIMESTAMP,
  server_error_count INT,
  server_last_error TEXT,
  uptime_sec BIGINT,
  free_heap INT,
  firmware_version VARCHAR(50),
  config_version INT,
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE device_connection_events (
  id UUID PRIMARY KEY,
  device_id UUID REFERENCES devices(id),
  event_type VARCHAR(100),
  target VARCHAR(50),
  level VARCHAR(20),
  message TEXT,
  detail JSONB,
  started_at TIMESTAMP,
  resolved_at TIMESTAMP,
  duration_sec INT,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE alerts (
  id UUID PRIMARY KEY,
  device_id UUID REFERENCES devices(id),
  signal_id UUID REFERENCES signals(id),
  alert_type VARCHAR(100),
  level VARCHAR(20),
  value DOUBLE PRECISION,
  message TEXT,
  status VARCHAR(30),
  started_at TIMESTAMP,
  ended_at TIMESTAMP,
  ack_by UUID,
  ack_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE audit_logs (
  id UUID PRIMARY KEY,
  user_id UUID,
  action VARCHAR(100),
  target_type VARCHAR(100),
  target_id UUID,
  before_data JSONB,
  after_data JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);
```
