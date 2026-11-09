# Collector Implementation Plan

Status: proposed · Owner: Srinivasan Vasudevan · Last updated: 2026-11-08

This plan turns `design.md`, `decision.md` and `roadmap_for_llm_agents.md` into a concrete
code layout, data model and task list for the **data ingestion (collector)** part of telegct.
Where it conflicts with those documents, this file wins (see §9).

**Product sequencing:** (1a) an observability dashboard showing current state plus days to
weeks of history, then (1b) the spatial heatmap, then (2) scheduler intelligence. Everything in
Phase 1 is optimised for (1a).

---

## 1. Language decision: Python first, Go only where it earns its place

| Criterion | Python (asyncio) | Go |
| :--- | :--- | :--- |
| Fit for 5–10 min polling, I/O-bound | Excellent; asyncio handles thousands of concurrent polls | Excellent, but the speed advantage is irrelevant at this cadence |
| Matches existing decisions | Yes: `decision.md` already picks `asyncio` + `asyncpg`, and the API uses FastAPI | Diverges from the design docs |
| One language for collectors, API, seed scripts and tests | Yes | No |
| SNMP | `pysnmp` 7.x (lextudio, asyncio-native): good | `gosnmp`: best in class |
| Redfish / HTTPS | `httpx`: good | `net/http`: good |
| Prometheus text format | `prometheus_client.parser`: good | `prometheus/common/expfmt`: good |
| gRPC / gNMI streaming | `grpcio`: works, but streaming at scale is heavier | Native; `gnmic` and openconfig tooling are Go-first |
| Container size | ~120–180 MB (slim) | ~15–25 MB (distroless static) |
| LLM-agent and team velocity | Highest | Good, but more verbose |
| Reuse of the old `/work/collector` code | Patterns are ported | Code is reused directly |

**Recommendation: Python 3.12 is the main language.** Every Phase-1 collector (CDU/SNMP,
PDU and BMC/Redfish, GPU exporters, vLLM) is a low-frequency, I/O-bound poll-parse-insert job.

**Use Go only when one of these is true:** (1) gRPC/gNMI or other long-lived high-rate
streaming; (2) a large fleet at sub-minute cadence, which is likely for the Phase-2
scheduler's vLLM feed at 10–30 s; (3) a node-local static-binary agent; (4) the best library
only exists in Go.

**Owner decision (2026-11-08): Go is the approved language for high-speed modules, for both
data ingestion and data query.** Python remains the default; Go is used per module where speed
or scale needs it:
- *Ingestion:* streaming (gRPC/gNMI), very large or sub-minute fleets (e.g. the Phase-2 vLLM
  feed at 10–30 s), node-local agents.
- *Query:* high-throughput read paths (e.g. a heatmap/scheduler query service over the `*_latest`
  views and caggs) if the Python FastAPI service becomes the bottleneck. It keeps the same API
  contract as `api/`.

Go modules are whole modules (never mixed with Python), emit the same `MetricSample` contract,
insert into the same tables, and reuse `collector/libs/go` (seeded from the old code, §2).
Each new Go module is added as a task in §5 with its justification (profiling result or scale
need).

---

## 2. Reusing the old code (`/Users/srinivasanvasudevan/work/collector`, Go)

The old repo is a clean, generic Go poller framework. It targets a different schema (InfluxDB
plus a `wastage_snapshots` table), so we port its **patterns** into Python now and keep its
**code** for the future Go library.

| Old file | What's worth keeping | Where it goes | Changes |
| :--- | :--- | :--- | :--- |
| `internal/config/config.go` | `${ENV}` expansion; auth/TLS/SNMP config schema; validation and defaults | `telegct_common/config.py` (pydantic) | **Fail fast on unset `${VAR}`.** The old code silently keeps the literal `${VAR}`. |
| `internal/auth/auth.go` | `Provider` (`inject`/`refresh`); None/Basic/Bearer/APIKey/OAuth2 with token cache and expiry skew | `protocols/http/auth.py` | 1:1 port. Add **Redfish session auth** (`X-Auth-Token`). |
| `internal/poller/job.go` | Connect backoff (5 s → 5 min); healthy/degraded/failed state machine; per-poll timeout; first poll runs immediately | `telegct_common/runner.py` | State is tracked per target. |
| `internal/poller/orchestrator.go` | One task per source | `runner.py` | No global factory registry: one collector type per container. |
| `rest/prometheus.go` `buildHTTPClient` | TLS CA bundle, mTLS, `insecure_skip_verify` | `protocols/http/client.py` | Same options. |
| `rest/prometheus.go` 401 refresh-retry | Refresh auth and retry once on 401 | `protocols/http/client.py` | Generalised to every HTTP call. |
| `rest/prometheus.go` vendor map (DCGM names) | Starting point for the **NVIDIA DCGM profile** (Vera Rubin) | `gpu-telemetry/profiles/nvidia-dcgm.yaml` | Verify the names against the deployed exporter. The AMD `rocm_gpu_*` names are wrong; use the AMD device-metrics-exporter names. |
| `rest/prometheus.go` PromQL query mode | Querying a central Prometheus | `sources/prometheus_query.py`, **deferred** | Phase 1 scrapes `/metrics` directly. |
| `writer/stdout_writer.go` | Dry-run sink | `sinks/stdout.py` | — |
| `writer/writer.go` | Influx write path, kept as a reference for a possible Phase-2 lake exporter (§7) | — | Not used in Phase 1. |
| `normaliser/` (wastage, cost, TDP) | Possibly a later dashboard metric | — | Not in Phase 1. |

For Go reuse, when the first Go module is approved, copy `auth`, `config`, `poller` and the
HTTP-client helper into `collector/libs/go` (module
`github.com/srivasud/telegct/collector/libs/go`). Note that the old `go.mod` says 1.25, but
the local toolchain is 1.23.

---

## 3. Code organisation

### 3.1 Improvements over a flat `collector/<module>` layout

1. **Three layers.** `protocols` (transport: HTTP, SNMP, gRPC, Modbus) → `sources` (protocol
   semantics: Redfish walking, Prometheus text parsing, SNMP OID tables) → collector modules
   (map readings to topology entities). Redfish serves both PDUs and BMCs, and the Prometheus
   parser serves both GPU exporters and vLLM.
2. **Vendor differences are data.** CDU OID maps, Redfish paths and exporter metric maps
   (AMD vs NVIDIA) live in `profiles/*.yaml`.
3. **One internal contract, `MetricSample`.** Collectors never write SQL. The Postgres sink
   routes samples by `entity_type` to the right table and pivots metric names to columns (§4).
4. **Shared build plus per-module Dockerfiles.** Build context is `collector/`, so each image
   copies the common lib.
5. **Simulators next to the code.** Fake SNMP agent, Redfish server and `/metrics` endpoints
   under `collector/testing/` serve unit tests, integration tests and compose dev runs.
6. **The schema is a shared contract at the repo root (`db/`).** Grafana dashboards are
   provisioned as code (`deploy/grafana/`).

### 3.2 Target layout

```
telegct/
├── CLAUDE.md
├── .github/copilot-instructions.md
├── designs/
├── db/
│   ├── migrations/                  # dbmate, plain SQL, up/down
│   └── seed/seed_topology.py
├── collector/
│   ├── pyproject.toml               # uv workspace root
│   ├── uv.lock
│   ├── libs/
│   │   ├── python/telegct_common/
│   │   │   └── src/telegct_common/
│   │   │       ├── models.py        # MetricSample, EntityType
│   │   │       ├── config.py        # YAML + ${ENV} + pydantic
│   │   │       ├── settings.py      # env-only settings (PG*, TELEGCT_*)
│   │   │       ├── runner.py        # interval loop, per-target state, backoff, semaphore
│   │   │       ├── logging.py
│   │   │       ├── health.py        # heartbeat file for Docker HEALTHCHECK
│   │   │       ├── topology.py      # resolve exporter labels → gpu_id / node_id
│   │   │       ├── sinks/{postgres.py, stdout.py, tables.py}  # tables.py = entity → table/column map
│   │   │       ├── protocols/{http/, snmp/, grpc/*, modbus/*}  # * deferred
│   │   │       └── sources/{redfish.py, prometheus_text.py, snmp_table.py}
│   │   └── go/                      # only once a Go module is approved
│   ├── cdu-collector/               # → cdu_metrics
│   ├── pdu-collector/               # → pdu_metrics, bmc_metrics (Redfish)
│   ├── gpu-telemetry/               # → gpu_metrics; profiles: amd-device-metrics.yaml, nvidia-dcgm.yaml
│   ├── vllm-collector/              # → inference_metrics
│   │   (each module: pyproject.toml, Dockerfile, config.example.yaml, .env.example,
│   │    profiles/, src/<pkg>/{__main__.py, collector.py}, tests/)
│   └── testing/{simulators/, fixtures/}
├── api/                             # FastAPI spatial heatmap (Phase 1b)
└── deploy/
    ├── docker-compose.yml           # timescaledb, migrate, simulators, collectors, grafana
    ├── env/*.env.example
    └── grafana/{provisioning/, dashboards/*.json}
```

Naming: folders are kebab-case and packages are snake_case. "gpu-telemetrics" is spelled
`gpu-telemetry`.

**Dependency direction:** `collector module → sources → protocols → third-party`. Modules
never import `httpx`/`pysnmp`/`grpc`/`asyncpg` directly, and never import each other
(enforced by `import-linter`).

### 3.3 Runtime model (each container)

```
load settings (env) + config.yaml (${ENV} expanded, validated)
connect asyncpg pool; load topology; drop targets whose id isn't in topology (warn)
loop every interval_seconds (default 300):
    gather(collect(target) for target in targets)    # Semaphore(max_concurrency), per-target timeout
    samples (real scrape timestamp per target) → validate → sink.insert(samples)
    update heartbeat; log a cycle summary (ok / failed / duration)
```

- **Append-only.** Rows are inserted only, never updated. `ON CONFLICT DO NOTHING` on the
  table PK makes retries idempotent. Each target's samples carry that target's scrape time.
- A failing target never fails the cycle. Per-target state and backoff are ported from `job.go`.
- Concurrency is capped per collector (default 16; BMCs are fragile).
- If the DB is down, retry with backoff while holding at most N cycles in memory, then drop the
  oldest and log the drop.
- **Counters** (e.g. `vllm:generation_tokens_total`) are stored as raw totals. Rates are
  computed at query time, so collectors stay stateless across restarts.
- `TELEGCT_DRY_RUN=true` switches to the stdout sink. On SIGTERM, finish the in-flight insert,
  then exit.

### 3.4 Configuration and secrets

- **Env (`.env`, never committed):**
  - DB: `PGHOST PGPORT PGDATABASE PGUSER PGPASSWORD PGSSLMODE` (asyncpg reads them natively).
  - Runtime: `TELEGCT_CONFIG TELEGCT_LOG_LEVEL TELEGCT_DRY_RUN`.
  - Device credentials: `CDU_SNMP_COMMUNITY`, `CDU_SNMPV3_*`, `PDU_REDFISH_USERNAME/PASSWORD`,
    `BMC_REDFISH_USERNAME/PASSWORD`, `GPU_EXPORTER_BEARER_TOKEN`.
- **`config.yaml`** (mounted; `.example` committed) lists targets as
  `{id, host, profile, credentials: ${VAR}}`. The `id` must exist in the topology tables.
  Phase 1 keeps target lists in config, per `decision.md`. Moving the inventory into the DB is
  a later option.
- **DB roles:** `telegct_ingest` has `INSERT` on metric tables and `SELECT` on topology.
  `telegct_reader` (Grafana, API) has `SELECT` only.

### 3.5 Docker

```dockerfile
# collector/cdu-collector/Dockerfile — build: docker build -f collector/cdu-collector/Dockerfile collector/
FROM ghcr.io/astral-sh/uv:0.5-python3.12-bookworm-slim AS build
WORKDIR /src
COPY pyproject.toml uv.lock ./
COPY libs/python libs/python
COPY cdu-collector cdu-collector
RUN uv sync --frozen --no-dev --package cdu-collector --no-editable

FROM python:3.12-slim-bookworm
RUN useradd -r -u 10001 telegct
COPY --from=build /src/.venv /app/.venv
ENV PATH=/app/.venv/bin:$PATH PYTHONUNBUFFERED=1
USER telegct
HEALTHCHECK --interval=60s CMD python -m telegct_common.health --max-age 900
ENTRYPOINT ["python", "-m", "cdu_collector"]
```

Run with `--env-file deploy/env/cdu-collector.env` and the config mounted at
`/etc/telegct/config.yaml`.

### 3.6 Tooling

`uv` workspace (`brew install uv`), `ruff`, `mypy --strict` on `libs/`, `pytest` +
`pytest-asyncio`, `respx`, `testcontainers[postgres]` with `timescale/timescaledb`, `dbmate`,
`import-linter`, and Grafana (provisioned).

Libraries: `httpx`, `pysnmp>=7`, `asyncpg`, `pydantic>=2`, `pyyaml`, `structlog`,
`prometheus-client` (parser only).

---

## 4. Data model (decided 2026-11-08; supersedes `design.md` §3)

**Principles**
- **Append-only**, with **one hypertable per level**. Each table has typed columns plus an
  `extra JSONB` column for vendor-specific values that don't have a column yet.
- **Real scrape timestamps.** There is no cross-collector merging.
- **TimescaleDB is the only store in Phase 1.** InfluxDB is not used (rationale in §7).
- **One topology model covers both deployment types.** A **machine** is the serving and
  scheduling unit: the whole rack for Helios and Vera Rubin, or one server for Instinct.
  - rack-scale: cabinet → machine (1 per rack) → node(s); GPU → tray → cabinet, and GPU → node;
  - Instinct: cabinet → machine (1 per server) → node (1:1); GPU → node (no tray).

  Per the owner (2026-11-08), a Helios rack of 16 trays is **one OS host**, i.e. one node per
  machine. The model also allows N nodes per machine (one OS per tray) with only a seed change.
  Verify this per platform before C1.

### 4.1 Topology

```sql
datacenter_zones(zone_id PK, name, location_description)
datacenter_rows(row_id PK, zone_id FK, row_name)
cabinets(cabinet_id PK, row_id FK, platform TEXT,             -- 'helios' | 'vera_rubin' | 'instinct'
         elevation_units INT DEFAULT 42, max_power_kw NUMERIC(6,2))
trays(tray_id PK, cabinet_id FK, u_position INT)              -- rack-scale only
machines(machine_id PK, cabinet_id FK NOT NULL,               -- vLLM / scheduling unit
         platform TEXT NOT NULL CHECK (platform IN ('helios','vera_rubin','instinct')),
         u_position INT NULL)                                 -- Instinct server position
nodes(node_id PK, hostname TEXT UNIQUE NOT NULL, machine_id FK NOT NULL)   -- OS hosts
gpus(gpu_id PK, node_id FK NOT NULL, tray_id FK NULL,         -- tray set for rack-scale
     gpu_index INT NOT NULL, vendor TEXT, model TEXT, serial TEXT NULL, uuid TEXT NULL,
     pci_address TEXT, vram_capacity_gb NUMERIC(6,2), UNIQUE (node_id, gpu_index))
bmcs(bmc_id PK, machine_id FK NOT NULL, tray_id FK NULL)       -- one per tray (rack-scale) or per server
cdus(cdu_id PK, name, row_id FK NULL)                         -- row-level or in-rack
cdu_cabinets(cdu_id FK, cabinet_id FK, PRIMARY KEY (cdu_id, cabinet_id))
pdus(pdu_id PK, cabinet_id FK NOT NULL, feed CHAR(1) NULL CHECK (feed IN ('A','B')))
```

GPU matching (`topology.py`): match on exporter `uuid` or `serial` when present, otherwise on
`(hostname, gpu_index)`. Both AMD and DCGM exporters label hostname and GPU index. The
GPU → tray assignment comes from the seed/inventory (by serial or PCI address), because with
one OS host per rack the hostname alone can't identify the tray.

### 4.2 Metric hypertables

| Table | PK | Typed columns (initial) | Writer |
| :--- | :--- | :--- | :--- |
| `cdu_metrics` | `(time, cdu_id)` | `supply_temp_celsius, return_temp_celsius, delta_t_celsius, flow_lpm, pressure_kpa, pump_speed_pct` | cdu-collector |
| `pdu_metrics` | `(time, pdu_id)` | `power_watts, energy_kwh, current_amps, voltage_volts` (outlets go in `extra`) | pdu-collector |
| `bmc_metrics` | `(time, bmc_id)` | `power_watts, inlet_temp_celsius, outlet_temp_celsius` | pdu-collector (Redfish BMC) |
| `gpu_metrics` | `(time, gpu_id)` | `util_pct, edge_temp_celsius, junction_temp_celsius, hbm_temp_celsius, power_watts, vram_used_gb, vram_total_gb` | gpu-telemetry |
| `inference_metrics` | `(time, machine_id, instance_id)` | `model, kv_cache_usage_ratio, num_requests_waiting, num_requests_running, prompt_tokens_total, generation_tokens_total` | vllm-collector |

`bmc_metrics` is the **tray-level** table: one BMC per tray in rack-scale systems, one per
server for Instinct. `inference_metrics` is the **machine-level** (Helios/Vera Rubin rack)
table. Tray, machine and cabinet views are joins. Every table also has
`extra JSONB` and `source TEXT` (collector and profile).

### 4.3 Storage policies (sized for "now plus a few weeks")

- Chunk interval: 1 day. Compression after 2 days, `segmentby` the entity id.
- **Raw retention: 35 days** (tunable).
- **Hourly continuous aggregates** (avg/min/max) per metric table, retained for 400 days, so
  week- and month-range charts stay fast.
- **`*_latest` views:** `DISTINCT ON (entity) ... ORDER BY entity, time DESC` limited to the
  last 3 intervals. They feed "current state" panels and the heatmap API.
- Index on `(entity_id, time DESC)` per table (the PK order already covers time-range scans).

### 4.4 Dashboards (Phase 1a)

Grafana uses its PostgreSQL datasource with `telegct_reader`. Queries hit raw tables for
ranges up to 7 days and the hourly caggs beyond that, using Grafana `$__timeGroup` macros.
Starter dashboards, provisioned from `deploy/grafana/dashboards/`:
1. Fleet overview: total power, hottest GPUs, CDU status, inference queue.
2. Cabinet drill-down.
3. Node and GPU drill-down.
4. Cooling (CDU).
5. Inference (vLLM).

---

## 5. Task list for coding agents (collector track)

Each task is one PR with tests. Don't start a task until its dependencies are merged.

| ID | Task | Depends on | Done when |
| :--- | :--- | :--- | :--- |
| C0 | Repo bootstrap: uv workspace, ruff/mypy/pytest, pre-commit, CI | — | CI green on an empty test suite |
| C1 | `db/migrations/0001_*`: topology (§4.1), metric hypertables (§4.2), compression, retention, caggs, `*_latest` views (§4.3), roles. `db/seed/seed_topology.py`: 1 zone, 2 rows, 4 cabinets (2 Helios racks of 16 trays as one machine/host each, 2 Instinct cabinets of 8-GPU servers), BMCs, GPUs, CDUs, PDUs | C0 | Up and down work on the timescaledb container; seed is idempotent |
| C2 | `telegct_common` core: models, config (port of `config.go`, fail on unset env), settings, logging, health, stdout sink | C0 | Unit tests for env expansion and validation |
| C3 | `runner` (port of `job.go`: per-target state, backoff, semaphore, timeouts, graceful shutdown) | C2 | Fake-collector tests: timeout, partial failure, shutdown |
| C4 | `sinks/postgres` (route by entity_type, pivot to columns, unknown → `extra`, batched insert with `ON CONFLICT DO NOTHING`, DB-down buffer) + `topology` resolver | C1, C2 | testcontainers test: rows land in the right tables, and a retry doesn't duplicate |
| C5 | `protocols/http` (port of `auth.go` + TLS + 401 retry) and `sources/prometheus_text` | C2 | respx tests per auth type |
| C6 | `gpu-telemetry` + AMD and DCGM profiles + metrics simulator + fixtures + Dockerfile | C3–C5 | Dry run is sane; integration writes 64 GPU rows |
| C7 | `vllm-collector` (instance → machine mapping in config) | C6 | Inference rows written; counters stored raw |
| C8 | `sources/redfish` (session auth, PowerSubsystem → Power fallback) + `pdu-collector` (PDU and BMC) + Redfish simulator | C5 | `pdu_metrics` and `bmc_metrics` rows written |
| C9 | `protocols/snmp` (v2c/v3) + `sources/snmp_table` + `cdu-collector` + SNMP simulator + one vendor profile | C3, C4 | `cdu_metrics` written; delta-T computed |
| C10 | `deploy/docker-compose.yml`: full stack with simulators; e2e smoke test | C6–C9 | `docker compose up` populates every table within one cycle |
| C11 | Grafana provisioning and the five starter dashboards (§4.4) | C1, C10 | Dashboards render from simulator data for 1 h, 7 d and 30 d ranges |
| — | Deferred: `protocols/grpc`, `protocols/modbus`, PromQL source, `libs/go`, `lake-exporter` (§7.1) | — | Built only when a consumer exists |

The API track (spatial heatmap, Phase 1b) can start after C1 and C4 and lives in `api/`. It
reads the `*_latest` views and the caggs for its time slider.

---

## 6. Open decisions

1. Confirm the OS-host count per rack for Helios and Vera Rubin (assumed 1; §4). This only
   affects the seed.
2. Data lake interface details (§7.1): deferred to Phase 2.
3. Module name: `gpu-telemetry` (assumed).

Resolved 2026-11-08: append-only per-level tables; vLLM gets its own table at **machine**
level (the whole rack for Helios and Vera Rubin); CDU and PDU tables are added; GPU mapping goes
through tray (rack-scale) or node (Instinct); TimescaleDB only for Phase 1; the historical lake
is pluggable, defaulting to TimescaleDB, with a customer-supplied client such as Snowflake (§7.1).

---

## 7. Phasing and storage evolution

| Phase | Deliverable | Storage implications |
| :--- | :--- | :--- |
| 1a | Collectors + TimescaleDB + Grafana observability (now plus days to weeks) | 35 days raw and 400 days hourly in Timescale; no other store |
| 1b | Spatial heatmap API + UI with a time slider | Reads `*_latest` and the caggs |
| 2 | Scheduler intelligence | vLLM cadence of 10–30 s (possibly a Go collector); a "current state" view joined with topology; a **lake exporter** job that copies new rows per table, using a per-table time watermark, to the long-term store |

**Why no InfluxDB in Phase 1:**
- The volume is easily handled by Timescale (~43M points/day at 10k GPUs).
- Dashboards and the scheduler need topology joins, which Influx can't do (`decision.md`
  already rejected it for this).
- A hot/cold split across two Influx instances would force every dashboard to stitch two
  datasets.

Grafana never needs the lake for its ranges (35 days raw plus 400 days hourly live in the
operational DB), so dashboards never stitch datasets.

### 7.1 Pluggable historical lake (Phase 2; details later)

- **`lake-exporter`** is a separate module/container. It reads new rows from each operational
  metric table using a per-table time watermark, then writes them through a
  **`LakeClient` interface**. Collectors are unaware of it.
- The interface is roughly: `ensure_schema(version)`, `write_batch(table, rows)`,
  `get_watermark(table)` and `commit_watermark(table, ts)`. Writes are idempotent on the table
  PK, so a crash or replay never duplicates rows.
- **Default client:** `timescale_client.py`, a second TimescaleDB instance with the same tables.
  It can be turned off via config.
- **Customer clients:** e.g. `snowflake_client.py`, implementing the same interface and
  selected by config (`lake.client: snowflake`), with credentials from env.
- **Published data schema:** versioned table definitions under `db/lake_schema/v<N>/`,
  generated from the migrations. This is the contract customers build their tables from.
  Breaking changes bump `N`.

---

## 9. Deviations from earlier design docs

- Folder layout: `collector/<module>` + `collector/libs` + `db/` + `api/` + `deploy/` replaces
  the roadmap's `telemetry-pipeline/`, `collectors/`, `config/`, `tests/`.
- The `design.md` §3 single wide `telemetry_metrics` table is replaced by per-level
  append-only hypertables (§4). `gpus` hangs off `nodes` instead of `trays`.
- Roadmap Task 2: `collect() -> Dict[str, Any]` becomes
  `async def collect(target) -> list[MetricSample]`.
- Roadmap Task 3 uses `httpx` instead of `aiohttp` (simpler API, `respx` mocking). The GPU
  collector supports AMD **and** NVIDIA DCGM exporters (Vera Rubin).
- Roadmap Task 4: one process per collector container, not a single `main.py`.
- The seed changes from 16 trays and 64 GPUs on one platform to a mixed Helios + Instinct
  topology (C1).
