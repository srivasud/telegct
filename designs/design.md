# System Architecture & Design Specification: Telemetry & Heatmap Platform

## 1. System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             PHYSICAL HARDWARE LAYER                              │
│  [Cooling (CDU)]    [Power (PDU/BMC)]    [AMD GPUs (amd-smi)]    [vLLM Inference]│
└───────┬─────────────────────┬─────────────────────┬──────────────────────┬───────┘
        │ SNMP / Modbus       │ HTTPS / Redfish     │ Prometheus Exporter  │ /metrics
        ▼                     ▼                     ▼                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         PLUGGABLE TELEMETRY INGESTION                            │
│  - Asyncio Collector Engine (Configurable polling loops: 5-10 mins)               │
│  - Protocol Adapters: SNMPCollector, RedfishCollector, PrometheusCollector        │
└─────────────────────────────────────┬────────────────────────────────────────────┘
                                      │ Batch Upsert / TimescaleDB
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           POSTGRESQL / TIMESCALEDB LAYER                         │
│  - Topology Mapping (Rows, Cabinets, Trays, GPUs)                                │
│  - Telemetry Hyper-table (High-frequency time-series storage)                     │
└─────────────────────────────────────┬────────────────────────────────────────────┘
                                      │ REST API / WebSockets
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             PHASE-1 HEATMAP ENGINE                               │
│  - Spatial Grid View (Rack Elevations, Tray U-Positions, GPU Sockets)            │
│  - Consolidated Health Score & Thermal / Power / Cache Color Encoding            │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Telemetry Sources & Protocol Interfaces

The platform ingests metrics from four distinct hardware and software abstraction boundaries:

| Layer | Source / Device | Protocol | Target Metrics Collected |
| :--- | :--- | :--- | :--- |
| **Cooling** | Coolant Distribution Unit (CDU) | SNMP v2c / v3 or REST | Supply temp ($^\circ\text{C}$), Return temp ($^\circ\text{C}$), Flow rate (LPM), Delta-$T$. |
| **Power** | Cabinet Rack PDU / Tray BMC | HTTPS Redfish REST API | Facility power feed (kW), Cabinet draw (W), Tray socket power (W). |
| **Hardware** | AMD Instinct GPUs (MI300X/MI300A) | Prometheus (`amd-smi` backend) | Edge/Junction temp ($^\circ\text{C}$), HBM temp ($^\circ\text{C}$), Board power (W), VRAM used (GB). |
| **Runtime** | vLLM Engine Instances | HTTP Prometheus `/metrics` | `vllm:kv_cache_usage_perc`, `vllm:num_requests_waiting`, active models. |

---

## 3. Database Schema Specification (`database.sql`)

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. Topology Hierarchy (Spatial / Geolocation Map)
CREATE TABLE datacenter_zones (
    zone_id VARCHAR(64) PRIMARY KEY,
    name VARCHAR(128) NOT NULL,
    location_description TEXT
);

CREATE TABLE datacenter_rows (
    row_id VARCHAR(64) PRIMARY KEY,
    zone_id VARCHAR(64) REFERENCES datacenter_zones(zone_id),
    row_name VARCHAR(64) NOT NULL
);

CREATE TABLE cabinets (
    cabinet_id VARCHAR(64) PRIMARY KEY,
    row_id VARCHAR(64) REFERENCES datacenter_rows(row_id),
    elevation_units INT DEFAULT 42,
    max_power_kw NUMERIC(6,2) NOT NULL
);

CREATE TABLE trays (
    tray_id VARCHAR(64) PRIMARY KEY,
    cabinet_id VARCHAR(64) REFERENCES cabinets(cabinet_id),
    u_position INT NOT NULL,
    bmc_ip VARCHAR(45) NOT NULL
);

CREATE TABLE gpus (
    gpu_uuid VARCHAR(128) PRIMARY KEY,
    tray_id VARCHAR(64) REFERENCES trays(tray_id),
    pci_address VARCHAR(32) NOT NULL,
    vram_capacity_gb NUMERIC(6,2) NOT NULL
);

-- 2. Time-Series Telemetry Table
CREATE TABLE telemetry_metrics (
    time TIMESTAMPTZ NOT NULL,
    entity_type VARCHAR(32) NOT NULL, -- 'cdu', 'cabinet_pdu', 'tray', 'gpu'
    entity_id VARCHAR(128) NOT NULL,

    -- Thermal & Cooling Metrics
    temp_celsius NUMERIC(5,2),
    cdu_supply_temp_c NUMERIC(5,2),
    cdu_return_temp_c NUMERIC(5,2),
    coolant_flow_lpm NUMERIC(6,2),

    -- Power Metrics
    power_watts NUMERIC(8,2),

    -- Memory & Cache Metrics (vLLM / Hardware)
    vram_used_gb NUMERIC(6,2),
    vram_capacity_gb NUMERIC(6,2),
    kv_cache_usage_perc NUMERIC(4,3),
    num_requests_waiting INT,
    active_models TEXT[],

    CONSTRAINT pk_telemetry PRIMARY KEY (time, entity_id)
);

CREATE INDEX idx_telemetry_entity_time ON telemetry_metrics (entity_id, time DESC);
```

---

## 4. REST API Endpoint Specifications for Heatmap

### Endpoint: `GET /api/v1/heatmap/spatial`
* **Query Parameters:** `zone_id` (optional), `timestamp` (optional, defaults to latest).
* **Response Body Structure:**
```json
{
  "timestamp": "2026-11-07T19:00:00Z",
  "zone_id": "zone-us-west-1",
  "rows": [
    {
      "row_id": "row-01",
      "cabinets": [
        {
          "cabinet_id": "cab-01",
          "elevation_units": 42,
          "max_power_kw": 100.0,
          "current_power_kw": 64.2,
          "trays": [
            {
              "tray_id": "tray-u12",
              "u_position": 12,
              "coolant_return_temp_c": 38.5,
              "gpus": [
                {
                  "gpu_uuid": "GPU-AMD-MI300X-01",
                  "temp_celsius": 68.5,
                  "power_watts": 450.0,
                  "vram_used_gb": 150.0,
                  "vram_capacity_gb": 192.0,
                  "kv_cache_usage_perc": 0.74,
                  "num_requests_waiting": 0,
                  "status": "GREEN"
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```
