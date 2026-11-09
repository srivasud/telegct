# Copilot instructions: telegct

Full rules are in `/CLAUDE.md` and the plan is in `/designs/collector_plan.md`. Read both
before making non-trivial changes. Summary:

- **Stack:** Python 3.12, asyncio, httpx, pysnmp 7, asyncpg, pydantic v2, structlog, uv
  workspace in `collector/`, PostgreSQL/TimescaleDB, FastAPI in `api/`. Go only for approved
  whole modules for high-speed data ingestion (gRPC/gNMI streaming, very large or sub-minute
  fleets, static node agents) and high-throughput data query paths (approved by the owner).
- **Layout:** `collector/libs/python/telegct_common` holds the shared code
  (`protocols/` = transport, `sources/` = protocol semantics, plus `runner`, `sinks`,
  `models`). Each `collector/<module>/` has its own `Dockerfile`, `config.example.yaml`,
  `.env.example`, `profiles/` and `tests/`. The schema lives only in `db/migrations/`.
- **Layering:** module → sources → protocols → third-party libs. Modules never import
  httpx/pysnmp/grpc/asyncpg directly, and never import each other.
- **Contract:** collectors return `list[MetricSample]`. Only `sinks/postgres.py` writes SQL,
  routing to the per-level hypertables (`cdu_metrics`, `pdu_metrics`, `bmc_metrics`,
  `gpu_metrics`, `inference_metrics`).
- **Append-only:** insert with `ON CONFLICT DO NOTHING`, never `UPDATE`. Use the real scrape
  timestamp. Store counters as raw totals.
- **Storage:** TimescaleDB only (no InfluxDB in Phase 1). Grafana reads raw tables for up to
  7 days and hourly continuous aggregates beyond that.
- **Vendor specifics** go in `profiles/*.yaml`, not code.
- **Secrets** only come from env vars (`PG*`, `PDU_REDFISH_*`, `CDU_SNMP_*`). YAML uses
  `${VAR}`, and an unset var is fatal. Never commit `.env` or log credentials.
- **Resilience:** timeouts on all I/O. One bad target never fails a cycle. Respect the
  concurrency limits.
- **Quality:** type hints, mypy strict on libs, ruff, pytest with fixtures and simulators;
  never real hardware. Missing values are `None`, not `0`. Put units in field names.
