# Implementation Roadmap & Step-by-Step Task Execution

This document contains discrete, sequential tasks for coding LLM agents to implement the **Infrastructure Telemetry & Heatmap Platform**.

---

## Task 1: Environment Setup & Database Schema Migration
* **Goal:** Initialize project structure and execute database migrations.
* **Requirements:**
  1. Create folder layout: `telemetry-pipeline/`, `collectors/`, `config/`, `tests/`.
  2. Implement `database.sql` creating physical topology tables (`datacenter_zones`, `datacenter_rows`, `cabinets`, `trays`, `gpus`) and time-series hyper-table `telemetry_metrics`.
  3. Write a seed script `scripts/seed_topology.py` to insert mock cluster topology (1 Zone, 2 Rows, 4 Cabinets, 16 Trays, 64 GPUs).

---

## Task 2: Implement Base Collector Interface & Config Parser
* **Goal:** Establish standard abstract class for telemetry adapters and parse YAML configuration.
* **Requirements:**
  1. Create `config.yaml` supporting configurable pipeline `interval_seconds` (300 to 600 seconds) and target lists for `cdu_devices`, `pdu_cabinets`, and `gpu_nodes`.
  2. Implement `collectors/base.py` containing abstract `BaseCollector` class with required `async def collect(self) -> Dict[str, Any]` interface.

---

## Task 3: Implement Protocol Adapters
* **Goal:** Build the three core collection adapters.
* **Requirements:**
  1. **SNMP Collector (`collectors/snmp_collector.py`):** Use `pysnmp` to poll CDU OIDs (`supply_temp`, `return_temp`, `flow_rate`).
  2. **Redfish HTTPS Collector (`collectors/redfish_collector.py`):** Use `aiohttp` to send authenticated GET requests to PDU Redfish `/redfish/v1/Power` endpoints and parse `PowerConsumedWatts`.
  3. **Prometheus Exporter Collector (`collectors/prometheus_collector.py`):** Use `aiohttp` to parse Node Exporter / AMD SMI and vLLM `/metrics` text endpoints to extract `power_watts`, `vram_used_gb`, `vllm:kv_cache_usage_perc`, and `vllm:num_requests_waiting`.

---

## Task 4: Pipeline Execution Engine & DB Writer
* **Goal:** Implement the main async orchestration loop.
* **Requirements:**
  1. Write `main.py` using `asyncio.gather()` to execute all configured collectors concurrently.
  2. Use `asyncpg` pool to perform batch `UPSERT` operations into `telemetry_metrics`.
  3. Respect the pipeline timer interval set in `config.yaml`.

---

## Task 5: Heatmap Spatial REST API Endpoint
* **Goal:** Provide REST API endpoint for the Phase-1 Heatmap UI.
* **Requirements:**
  1. Implement FastAPI endpoint `GET /api/v1/heatmap/spatial`.
  2. Execute SQL query joining spatial topology tables with the latest telemetry snapshot per entity.
  3. Compute status flags (`GREEN`, `YELLOW`, `RED`) based on thermal, power, and KV-cache thresholds.

---

## Task 6: Unit & Integration Testing
* **Goal:** Validate end-to-end telemetry flow.
* **Requirements:**
  1. Write mock hardware server fixtures for SNMP, Redfish, and Prometheus endpoints.
  2. Write integration tests verifying metric ingestion and correct database insertion.
