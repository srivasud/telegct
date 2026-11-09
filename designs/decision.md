# Phase-1 Scope & Architectural Decisions

## 1. Phase-1 Functional Scope

### Included in Phase 1
1. **Multi-Protocol Telemetry Collectors:**
   * **SNMP Adapter:** Polling CDU devices for supply/return temperature and fluid flow rates.
   * **Redfish HTTPS Adapter:** Polling cabinet PDUs and tray BMCs for aggregate power utilization.
   * **Prometheus Exporter Adapter:** Ingesting node-level `amd-smi` hardware metrics (temperature, power, VRAM) and vLLM `/metrics` (KV-cache utilization, waiting queue count).
2. **Configurable Pipeline Engine:**
   * Execution loops configurable via `config.yaml` with a default cadence of **5 minutes** (adjustable to 10 minutes).
   * Asynchronous non-blocking collection using Python `asyncio` and `asyncpg`.
3. **Data Persistence:**
   * Storing static physical geolocation topology (Zones, Rows, Cabinets, Trays, GPUs).
   * Persisting normalized time-series metric snapshots to PostgreSQL.
4. **Spatial Heatmap Backend Data Layer:**
   * REST API exposing full physical-to-logical hierarchy state for real-time heatmap rendering.

### Excluded from Phase 1 (Deferred to Future Phases)
* Direct hardware control feedback loops (e.g., auto-throttling GPUs or modulating CDU pump speeds).
* Dynamic live migration of vLLM cache blocks across trays (Phase-2 Scheduler integration).
* Dynamic parameter-driven weight footprint calculation via `config.json` parsing inside runtime containers.

---

## 2. Key Architectural Decisions Matrix

| Decision Area | Chosen Strategy | Reason / Trade-Off | Alternatives Considered |
| :--- | :--- | :--- | :--- |
| **GPU Metrics Provider** | AMD Exporter (`amd-smi` + Prometheus) | **AMD Fabric Manager** only configures XGMI/Infinity Fabric links and topology routing. Hardware metrics require `amd-smi`. | Direct C-bindings to ROCm SMI API (higher complexity). |
| **Runtime Metrics** | vLLM `/metrics` endpoint | Direct exposure of engine internals (`kv_cache_usage_perc`, `num_requests_waiting`). | Polling raw PyTorch CUDA/ROCm memory allocators (misses KV-cache context). |
| **Pipeline Cadence** | Batch Polling (5-10 min) | Fits facility infrastructure interfaces (SNMP/Redfish) without saturating BMC control planes. | Streaming WebSockets (excessive overhead for facility hardware). |
| **Storage Engine** | PostgreSQL (TimescaleDB) | Enables structured spatial joins (Rows $\rightarrow$ Cabinets $\rightarrow$ Trays $\rightarrow$ GPUs) alongside time-series metrics. | Pure Prometheus or InfluxDB (lacks rich relational geolocation mapping). |