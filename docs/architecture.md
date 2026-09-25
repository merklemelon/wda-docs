# WorldDominationApp — Development Architecture

**Version:** 1.3 (2026-09-24; derived from vision doc v0.4, 2026-08-15)
**Status:** living document. Bump the version when a structural decision changes.
**Audience:** Claude Code and the developer. This is the *how* at system level; the
module level is [micro-architecture.md](micro-architecture.md); the *why* is in
`docs/reference/worlddomination-architecture.md`.

Companion references: [fundamentals-and-valuation.md](fundamentals-and-valuation.md)
(data lineage), [thesis.md](thesis.md), [forward-analysis.md](forward-analysis.md),
[gics-sectors.md](gics-sectors.md), [verification.md](verification.md) (tests and risk),
[edgar-ingestion.md](edgar-ingestion.md), [decisions.md](decisions.md) (ADRs).

---

## 1. Shape of the system

One Python application, one Docker image, three process roles on fly.io, one Supabase
Postgres (pgvector) database.

| Process | Entry point | Count | Responsibility |
|---|---|---|---|
| `web` | `uvicorn wda.app:create_app_from_settings --factory` | 1 (kept warm: `min_machines_running = 1`) | FastAPI + the MCP endpoint (FastMCP) under `/mcp`, OAuth 2.1 / bearer auth. Stateless. |
| `worker` | `python -m wda.jobs.worker` | 1 | Claims jobs from the `jobs` table, runs registered handlers. Also where one-off CLIs run. |
| `scheduler` | `python -m wda.jobs.scheduler` | **exactly 1** (advisory-lock singleton) | Enqueues periodic work. Never does work itself. |

Deliberate omissions still hold: no Kubernetes, Airflow, Kafka, Redis, Celery,
microservices. Two were added only after a measured problem: a persisted L5 (factor
snapshots) when the on-demand screen wedged the web app, and a streaming zip reader when
a 990k-member archive thrashed the worker.

Data volumes (2026-09-16): 5,318 companies (all with SIC), ~70k filings, ~5.8M XBRL
facts (fiscal-year partitions), prices for 5,158 tickers (~3 years), 3,712 factor
snapshots, 33 enriched companies.

### 1.1 Layer model (import direction is enforced by import-linter)

```
L6  mcp        wda/mcp            40 tools + 3 prompts over services; shaping; OAuth
L5  scoring    wda/scoring        factor snapshots (persisted), screen, rules, cohorts
    thesis     wda/thesis         reverse-DCF, implied expectations, the thesis engine
L4  enrich     wda/enrich         LLM extraction (4 targets), embeddings, semantic search
L3  derived    wda/xbrl/metrics, wda/market   canonical financials, ratios, valuation,
                                  prices, price-derived risk (vol, drawdown)
    macro      wda/macro          statistical-agency series (FRED rates, EIA energy)
L2  normalize  wda/xbrl (catalog), wda/filings/sections, chunking
L1  raw        wda/filings, wda/xbrl/facts, wda/core/blobstore   immutable, point-in-time
L0  ingest     wda/ingest         EDGAR client, parsers, sync services, bulk CLIs, handlers
```

Enforced contract (`pyproject.toml`): `mcp > scoring > enrich > (xbrl | market) > filings
> ingest`. `universe`, `cohorts`, `thesis`, `macro`, `jobs` and `core` are unconstrained (`core` is
layer-agnostic infrastructure; `thesis` composes scoring/market/xbrl/enrich services;
`universe`/`cohorts` are plain stores). Sibling layers (`xbrl`, `market`) never import
each other — the composition root (`Container`) bridges them.

### 1.2 The known patterns it composes

Hexagonal ports & adapters inside a modular monolith; medallion layering (raw → normalized
→ derived → enriched → scored), each layer rebuildable from the one below; **bitemporal**
facts (`period_end` + `filed_at`, append-only, reads accept `as_of`); a CQRS-style read
model for L5 (the snapshot is a point-in-time feature store); a transactional job queue
(`FOR UPDATE SKIP LOCKED`); unstructured-to-structured extraction with a vector index;
an assertion-based monitoring layer (theses); a tool server for agents (MCP). See
`decisions.md` for the individual choices.

---

## 2. Repository layout (as it is)

```
wda/
├── app.py                 FastAPI factory; mounts /health and the MCP app
├── core/                  config (pydantic-settings, WDA_*), db, http (token bucket + retry),
│                          blobstore (Local | S3), errors, logging (structlog), clock, container
├── universe/              companies + tickers (models, repository), gics.py (SIC->GICS)
├── filings/               filings + documents, sections.py (Item 1/1A/7), chunking.py
├── xbrl/                  allowlist (43 concepts), metrics.py (catalog + series engine), facts repo
├── ingest/edgar/          client, parsers/, services/ (sync + bulk), bulk_archive, zipindex,
│                          universe_filter, handlers (5 job kinds), 2 bulk CLIs
├── market/                prices (models, repo), valuation.py, technicals.py (realised vol,
│                          drawdown, distance-from-high), service (sync), adapters/yahoo,
│                          handlers (2 job kinds), bulk_prices_cli
├── macro/                 statistical-agency series: ports, models, repo, service (sync),
│                          adapters/ (fred, eia), handlers (2 job kinds)
├── enrich/                extractors (4 TargetSpecs), service (locate + extract), budget,
│                          embed_service, embeddings port, adapters (anthropic, openrouter),
│                          repository (rows, token usage, two vector stores), handlers (3 kinds)
├── cohorts/               saved cohorts store (static + rule)
├── scoring/               factors, scorer, rules, cohorts (resolver), snapshots (pure),
│                          snapshot_builder, repository, service (screen), handlers (2), snapshot_cli
├── thesis/                reverse_dcf, implied (+ calibrate_cli), checks, schemas, engine,
│                          resolver, repository, service, handlers (1)
├── mcp/                   server (36 tools, 3 prompts, instructions), tools, shaping, auth, oauth
├── jobs/                  models, queue, registry, context, worker, scheduler
└── api/                   (empty — REST routers were never needed; MCP is the product)
alembic/versions/0001..0009
docs/  tests/{unit,integration,fixtures,builders.py,fakes.py}
```

### 2.1 Inside a domain package

`models.py` (SQLAlchemy, table shape only) · `schemas.py` (Pydantic boundary contracts) ·
`ports.py` (Protocols) · `repository.py` · `service.py` (use cases over ports; no SQL, no
HTTP) · `adapters/` · `handlers.py` (thin job handlers resolving services from the
container). Pure computation gets its own module (`metrics.py`, `valuation.py`,
`scorer.py`, `rules.py`, `snapshots.py`, `reverse_dcf.py`, `engine.py`, `gics.py`,
`zipindex.py`) — the modules with the highest unit coverage.

---

## 3. Design patterns and SOLID (unchanged in spirit, concrete names updated)

- **Repository** per aggregate; intention-revealing methods (`facts_for_concepts(cik,
  concepts, as_of)`, `latest_close(ticker, as_of)`, `factors_for(ciks)`).
- **Unit of Work** — services receive a `sessionmaker`, not a session.
- **Ports & Adapters** — `EdgarClient`, `MarketDataProvider` (Yahoo), `LlmClient`
  (Anthropic), `VectorEmbedder` (OpenRouter), `BlobStore` (local | S3), `Clock`,
  `RateLimiter`; tests inject fakes (`tests/fakes.py`).
- **Registry** — job handlers (`@job_handler("kind")`), enrichment targets
  (`EXTRACTORS`), metric catalog (`CATALOG`, `RATIOS`), factors (`FACTORS`), valuation
  defs (`VALUATIONS`), check vocabulary (a discriminated union), built-in cohorts.
- **Strategy** — concept fallback chains and derived metrics in the catalog; section
  extraction by item; the three reverse-DCF bases.
- **Specification** — cohort rules (`[field, op, value]` clauses) and thesis checks share
  one operator vocabulary (`rules.compare`).
- **Watermark / idempotent consumer** — every sync records a cursor per source (or per
  entity: `market.prices:<ticker>`); every write is an upsert on a natural key.
- **Circuit breaker** — the daily LLM budget (`BudgetGuard`, `token_usage` ledger).
- **Two-step writes** — every MCP tool that spends or writes previews first, then
  `confirm=true` (rule 9).
- **Composition root** — `wda/core/container.py` is the only place concrete classes meet.

Refused: service locators, ORM models with behaviour, vendor branches, broad `except`,
`datetime.now()` outside `Clock`, importing across the layer contract.

---

## 4. Core infrastructure

**Config.** `Settings` (`WDA_*`): database URL (Supavisor **session-mode** pooler — the
scheduler's advisory lock needs session semantics), EDGAR user agent + RPS (8), blob
backend/root, LLM daily budget (USD), Anthropic + OpenRouter keys and models, MCP
token / allowed hosts, OAuth issuer/client/signing key/redirects/scope, log format, and
the schedules (ticker sync interval, daily-index interval, reaper, price sync 18:00 ET
Mon–Fri, snapshot revalue 19:30 ET, thesis evaluation 20:00 ET).

**Database.** Async SQLAlchemy 2 over asyncpg, `pool_pre_ping`, `statement_cache_size=0`.
Alembic migrations 0001–0009, applied by the deploy's release step. `xbrl_facts` is
range-partitioned by `fy`. Point-in-time is a schema property: every fact row carries
`period_end` and `filed_at`; restatements append. Operating envelope (Supabase **Large**,
8 GB RAM, since 2026-09-17): `shared_buffers` 2 GB holds the working set, so per-company
fact reads and vector writes are sub-second; the conventions learned on the old tier
(estimates instead of exact counts over facts, set-based reads, batched vector writes,
a 35-day nightly price window) stay because they were amplifiers regardless. The
statement timeout is 2 min. Long-lived processes ride out a database restart
(`TRANSIENT_DB_ERRORS`): the worker backs off, the scheduler re-acquires its singleton
lock every minute. Direct (non-pooler) connections are used only for diagnosis.

**HTTP.** `RateLimitedClient`: token bucket at 8 req/s, gzip, retry with backoff on
429/5xx, structured logging. Bulk archive downloads have their own retry (`bulk_archive`).

**Blob store.** `edgar/<cik10>/<accession_nodash>/<filename>`; local in dev, S3 (MinIO in
CI, Tigris-compatible) in prod.

**Jobs.** `jobs` table (kind, payload, status queued|running|done|failed, priority,
attempts, run_after, locked_at/by, error). Claim with `FOR UPDATE SKIP LOCKED`; reaper
requeues stale `running` rows after 15 min; **no heartbeat**, so anything longer than
the TTL is a one-off CLI, not a job. Priorities: enrichment/fetch 0, snapshot refresh −5,
price fan-out −10. Registered kinds:

| Kind | Trigger | Does |
|---|---|---|
| `edgar.sync_tickers` | every N h | ticker→CIK map; primary-ticker rule |
| `edgar.sync_daily_index` | every N h | new filings from the SEC daily index (**failing since 09-12: asks for today's not-yet-published file; fix pending**) |
| `edgar.sync_submissions` / `edgar.sync_companyfacts` | `request_sync`, or the first step of a `request_enrichment` chain | profile + filings (forwards the chain); facts (then enqueues `scoring.snapshot`) |
| `edgar.fetch_filing` | `request_fetch` / index sync / the chain | 10-K to the blob store (forwards the chain on success) |
| `enrich.extract` / `enrich.embed` / `enrich.index_filing` | one `request_enrichment` confirm (extract → embed chained; index alongside) or the granular `request_*` repairs | LLM rows per target; vectors written in committed batches of 32, resumable |
| `market.sync_prices_all` → `market.sync_prices` | cron 18:00 ET | daily closes for every company with facts |
| `scoring.snapshot` | after a facts sync | one company's factor snapshot |
| `scoring.revalue_all` | cron 19:30 ET | every snapshot's multiples off the new closes |
| `thesis.evaluate_all` | cron 20:00 ET | re-check every thesis, record and diff |

One-off CLIs (run on the worker, never as jobs): `bulk_companyfacts_cli`,
`bulk_submissions_cli`, `bulk_prices_cli`, `scoring.snapshot_cli`, `thesis.calibrate_cli`.

---

## 5. Data model (23 tables, migrations 0001–0013)

```
0001  companies, tickers, filings, filing_documents, xbrl_facts (PARTITION BY RANGE fy),
      ingest_watermarks, jobs; pgvector enabled
0002  risk_themes, token_usage
0003  products, forward_statements, event_calendar; risk_themes.is_new
0004  enrichment_embeddings   (pgvector: distilled rows, cosine search)
0005  filing_chunks           (pgvector: 10-K passages by item)
0006  prices_daily            (ticker, date, close [raw], source; UNIQUE(ticker,date))
0007  cohorts, cohort_members (static and rule cohorts)
0008  factor_snapshots        (cik PK; quality, inputs, factors JSONB; price, price_date;
                               fundamentals_at, valued_at)
0009  theses, thesis_conditions, thesis_assumptions, thesis_evaluations
0010  prices_daily.adj_close  (split/dividend adjusted — the series returns are computed on)
0011  prices_daily.volume     (liquidity: average dollar volume)
0012  jobs expression index on payload->>'cik' (the per-company job trail)
0013  macro_series            (source, series_id, observed_at, value, vintage;
                               UNIQUE(source, series_id, observed_at))
```

`macro_series.vintage` is the date a value was *published*, distinct from the date it
describes. Macro figures are revised for years, so a point-in-time read excludes both
readings dated after `as_of` and readings not yet published by then — the same discipline
`filed_at` enforces on facts. It is nullable on purpose: an agency that publishes no
release date records None rather than the fetch date dressed up as one, and `as_of` then
honestly degrades to filtering on the observation date. See `docs/decisions.md`.

Lineage of every derived number is documented in
[fundamentals-and-valuation.md](fundamentals-and-valuation.md). L5 persistence (why the
snapshot exists, its three refresh cadences, the 50-company cap on point-in-time screens)
is §4.3 there and ADR-level in `decisions.md`.

---

## 6. The MCP surface (the product)

40 tools in ten groups plus three prompts (`diligence`, `screen_value`, `compare`), with
server instructions that carry driving guidance per feature (pinned by a test):

| Group | Tools |
|---|---|
| Find | `search_companies`, `get_company`, `list_concepts` |
| Read numbers | `get_financials`, `get_facts`, `list_filings`, `get_filing_document` |
| Read narrative | `get_enrichment`, `get_risk_themes` |
| Search by meaning | `search_enrichment`, `search_filings` |
| Value | `get_valuation`, `get_prices` |
| Rank | `screen`, `get_score`, `list_factors` |
| Cohorts | `list_cohorts`, `get_cohort`, `create_cohort`*, `delete_cohort`*, `get_rule_fields`, `list_sectors` |
| Forward | `implied_expectations` |
| Thesis | `get_thesis_schema`, `create_thesis`*, `get_thesis`, `evaluate_thesis`, `list_theses`, `delete_thesis`* |
| Extend / ops | `request_enrichment`* (one approval: sync → fetch → extract → embed → index), `get_pipeline_status`, `get_ingest_status` (with `recent_failures`), and the granular `request_sync`*, `request_fetch`*, `request_embed`*, `request_index`*, `request_price_sync`* |

\* two-step (preview → `confirm=true`). Responses are shaped (`DEFAULT_MAX_ROWS`,
pagination) so no tool can flood a context. Auth: OAuth 2.1 (fixed-client provider) for
claude.ai connectors, bearer token for direct clients. REST under `/api/` was never
built; `/health` is the only non-MCP route.

---

## 7. Testing

Measured in [verification.md](verification.md): 425 tests (370 unit, no network/DB; 55
integration on pgvector + MinIO with migrations from scratch); statement coverage 68%
unit-only, 85% combined; the pure cores at 91–99%. CI runs lint (ruff, mypy `--strict`,
import-linter) → unit → integration on every PR; `main` deploys only when green. The
risk register there is the honest list of what tests cannot see (live third parties,
scale, LLM quality, the deployed service).

---

## 8. Operational conventions (learned)

- Structured logs per job; `get_ingest_status` is the health readout (counts are
  estimates; snapshot freshness; per-source-family watermarks).
- The worker has no volume: `/tmp` is wiped on every deploy roll; long one-offs must be
  relaunched after a deploy, and the deploy's GHA success can precede the worker's final
  roll by ~2 min — confirm the new code is on the machine before launching.
- Never run `count(*)` or `DISTINCT cik` over `xbrl_facts` in a request path; never
  open `submissions.zip` with `zipfile`; run snapshot rebuilds outside 18:00–19:30 ET.
- The web machine is kept warm; a stopped web machine looks like "MCP is down" from
  claude.ai.
- Rebuild commands: `snapshot_cli` (L5 from L1/L3), `bulk_companyfacts_cli` /
  `bulk_submissions_cli` (L1 from EDGAR archives), `bulk_prices_cli`.
- Secrets only via fly secrets / `.env`; never printed.

---

## 9. Deployment path

GitHub Actions on `main`: lint + tests → `fly deploy --remote-only` (one image, three
process groups, `iad`) with a release step `alembic upgrade head` → fly rolls web, worker
and scheduler. Supabase Postgres in every environment except unit tests. The MCP
endpoint is reached by claude.ai through the WDA connector (OAuth) and by Claude Code
through the same connector.
