# CLAUDE.md: telegct

telegct is a datacenter telemetry and heatmap platform. Collectors poll CDUs (SNMP), PDUs and
BMCs (Redfish/HTTPS), AMD GPUs (amd-smi Prometheus exporter) and vLLM (`/metrics`). They write
normalised snapshots into PostgreSQL/TimescaleDB, and a FastAPI service serves a spatial
heatmap built on the zone → row → cabinet → tray → GPU topology.

**Read first:** `designs/collector_plan.md` (layout, task list, open decisions), then
`designs/design.md` (schema, API) and `designs/decision.md` (scope). If the roadmap and the
plan disagree, the plan wins. If the plan and this file disagree, ask the user.

## Repository layout

```
db/migrations/          dbmate SQL migrations: the ONLY way to change the schema
db/seed/                topology seed scripts
collector/              uv workspace for all ingestion code
  libs/python/telegct_common/   shared lib: models, config, runner, sinks, protocols, sources
  libs/go/              shared Go lib (only once a Go module is approved)
  <name>-collector/     one deployable module per source family, each with its own Dockerfile
  gpu-telemetry/        amd-smi exporter scraping
  testing/              simulators (SNMP, Redfish, /metrics) and recorded fixtures
api/                    FastAPI heatmap service
deploy/                 docker-compose, env examples, grafana/ (provisioned dashboards as code)
```

Topology: a **machine** is the vLLM/scheduling unit (a whole Helios or Vera Rubin rack, or one Instinct server). Rack-scale: GPU → tray → cabinet and GPU → node → machine. Instinct is
GPU → node → cabinet. TimescaleDB is the only store in Phase 1 (no InfluxDB). Phase 1a is the
Grafana observability dashboards, Phase 1b the spatial heatmap API, and Phase 2 the scheduler.

## Architecture rules (do not break these)

1. **Layering:** `collector module → sources → protocols → third-party libs`.
   - `protocols/` = transport only (http, snmp, grpc, modbus). No device semantics.
   - `sources/` = protocol semantics (Redfish walking, Prometheus text parsing, SNMP OID tables).
   - Collector modules map readings → `MetricSample` for topology entities.
   - Collector modules must NOT import `httpx`, `pysnmp`, `grpc` or `asyncpg` directly, and must
     not import each other.
2. **One contract:** collectors return `list[MetricSample]`. Only `sinks/postgres.py` writes
   SQL. It routes samples by entity type to the per-level hypertables (`cdu_metrics`,
   `pdu_metrics`, `bmc_metrics`, `gpu_metrics`, `inference_metrics`; plan §4).
3. **Vendor differences go in `profiles/*.yaml`**, not in `if vendor == ...` code.
4. **Append-only:** metric tables are insert-only (`ON CONFLICT DO NOTHING` for idempotent
   retries). Never `UPDATE` or merge rows. Each sample carries its target's real scrape
   timestamp. Store counters as raw totals; rates are computed at query time.
5. **Resilience:** every network call has a timeout. One failing target must never fail the
   cycle or crash the process. Respect the per-collector concurrency semaphore (BMCs are
   fragile). Don't poll more often than the configured interval (default 300 s).
6. **Async everywhere** in collectors. No blocking I/O in the event loop.
7. **Schema changes** only through a new numbered migration in `db/migrations/` with a working
   `down`. Never edit an applied migration.

## Secrets and config

- Credentials come ONLY from environment variables (`.env` files, gitignored). DB access uses
  `PGHOST PGPORT PGDATABASE PGUSER PGPASSWORD PGSSLMODE`. Device creds use names such as
  `PDU_REDFISH_PASSWORD` and `CDU_SNMP_COMMUNITY`.
- `config.yaml` references secrets as `${VAR}`. An unset variable is a startup error.
- Never commit `.env`, real IPs or hostnames, certificates, or recorded fixtures that contain
  credentials or serial numbers. Commit `*.example` files instead.
- Never log secrets, auth headers or full request bodies from BMCs.

## Language

- **Python 3.12 is the default** for collectors, the API and scripts.
- Go is allowed only for a whole module that needs it: gRPC/gNMI streaming, very large or
  sub-minute fleets, or static node agents. See plan §1. Go modules emit the same contract and
  reuse `collector/libs/go`. Ask before starting a Go module.

## Commands

```bash
cd collector
uv sync                                   # install workspace
uv run ruff format . && uv run ruff check . --fix
uv run mypy libs/python
uv run pytest                             # unit tests
uv run pytest -m integration              # needs Docker (testcontainers timescaledb)
TELEGCT_DRY_RUN=true TELEGCT_CONFIG=gpu-telemetry/config.example.yaml uv run python -m gpu_telemetry
docker build -f collector/gpu-telemetry/Dockerfile collector/      # run from repo root
docker compose -f deploy/docker-compose.yml up                     # full stack with simulators
```

## Coding conventions (Python)

- Type hints everywhere. `mypy --strict` must pass on `libs/`. Use pydantic v2 for config and
  external payloads, and dataclasses or pydantic for internal models.
- Use `structlog` with structured key/value logging. Include `collector`, `target` and `cycle`
  in the context. No `print`.
- Small pure functions for parsing and mapping. Keep I/O at the edges so parsers can be tested
  against fixtures.
- Units in names: `power_watts`, `temp_celsius`, `flow_lpm`. Convert units in the collector,
  never in the sink.
- Missing metric = `None` (stored as NULL), never `0`.

## Testing (definition of done)

- Every parser or mapper has a unit test against a recorded fixture in `collector/testing/fixtures/`.
- Every protocol client is tested against a simulator or mock (`respx` for HTTP, the SNMP
  simulator for SNMP). **Never hit real hardware in tests.**
- Sink and topology changes need a testcontainers integration test.
- A task is done when ruff, mypy and pytest pass, the module's dry run prints sane samples, and
  `config.example.yaml` and `.env.example` are updated.

## Working style for agents

- Do one task from `designs/collector_plan.md` §5 per PR or branch, and mention its ID
  (e.g. `C5`) in the commit message.
- Don't build speculative abstractions. gRPC, Modbus, PromQL-query and Go libs are deferred
  until something consumes them.
- If you hit an open decision (plan §6), stop and ask instead of picking silently.
- Old Go reference code lives at `/Users/srinivasanvasudevan/work/collector`. Port patterns
  from it (auth providers, backoff, env expansion), not its InfluxDB or wastage logic.
