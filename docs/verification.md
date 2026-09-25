# Verification — Test Strategy, Inventory, Coverage and Risk

**Status:** reference, measured 2026-09-16 on `main` (after #81)
**Related:** `CLAUDE.md` rules 2 (every module has unit tests) and 4 (point-in-time),
`docs/architecture.md` §7 (testing strategy), `.github/workflows/ci.yml`.

Numbers in this document were produced by running the suites, not estimated:
`pytest --collect-only` for the inventory; `pytest --cov=wda` for the unit run and for the
unit + integration run on a throwaway `pgvector/pgvector:pg16` container. Re-measure the
same way when this document is updated.

---

## 1. Strategy

Four principles, in the order they were adopted:

1. **Unit tests never touch the network or a database.** Every domain package depends on
   `Protocol` ports; unit tests inject in-memory fakes (`tests/fakes.py`), recorded EDGAR
   JSON (`tests/fixtures/edgar/`), a `FakeClock`, and `respx` for HTTP. A unit test that
   needs Postgres is a bug in the test.
2. **Pure functions carry the logic; I/O is a thin shell.** The metrics engine, the
   scorer, the rule predicate, valuation, the reverse-DCF, the snapshot maths, the thesis
   engine, the zip index — each is a module of pure functions with the heaviest unit
   coverage in the repo (94-99%). Repositories, services and job handlers wrap them and are
   covered by integration tests. When a bug is found in I/O code, the fix usually moves
   logic *out* of it (e.g. `fiscal_year_rows`, `summarize_sources`, `snapshot_row`).
3. **Integration tests exercise the real stack, seeded.** Postgres (pgvector) + MinIO in
   CI, Alembic migrations applied from scratch, real repositories, real services, the MCP
   tool functions called directly with a `Container`. Data is seeded from builders
   (`tests/builders.py`) and fixtures, never fetched. Marked `@pytest.mark.integration`,
   excluded from the default run, required by CI.
4. **Tests are the spec for behaviour that was wrong once.** Every production incident in
   the log became a test before the fix shipped: the reverse-DCF bracket that "solved" a
   $1B share price, the fiscal-year cap eaten by quarterly instants, the primary-ticker
   duplication, the null `exchanges`, the 990k-member zip, the bulk 429, the executemany
   UPDATE. The regression suite is largely a history of the system's real failures.

What is deliberately **not** in scope: live third-party APIs (EDGAR, Yahoo, Anthropic,
OpenRouter), the deployed MCP transport, performance under production data volumes, and
LLM output quality. §5 treats each as a named risk with a mitigation.

### Tooling

`pytest` + `pytest-asyncio` (auto mode), `pytest-cov`, `hypothesis` (property-based:
accession-number round-trips, the token-bucket limiter, EDGAR client retries, the explorer
script), `respx` (HTTP mocking for the EDGAR client, Yahoo adapter, OpenRouter embedder).
`filterwarnings = error` — any warning fails the run. `--strict-markers`. Lint gate:
`ruff` (lint + format), `mypy --strict` over `wda/`, `import-linter` for the layer
contract. All of it runs in CI on every PR; `main` deploys only after it passes.

---

## 2. What runs where

| Command | Runs | When |
|---|---|---|
| `make test` | 370 unit tests, no network, no DB, ~40 s | every PR (CI), every local iteration |
| `make test-int` | 55 integration tests against Postgres + MinIO, ~2 min | every PR (CI: pgvector service + MinIO container); locally via `docker run pgvector/pgvector:pg16` on a spare port + `WDA_TEST_DATABASE_URL` |
| `make lint` | ruff check + format check, mypy --strict, import-linter | every PR (CI), before every commit |
| CI job "Lint + tests" | lint → unit → integration | PR and `main` |
| CI job "Deploy to fly.io" | `fly deploy --remote-only`, release step `alembic upgrade head` | `main` only, after tests pass |

There is **no post-deploy smoke test** and **no coverage threshold** in CI today (§6).

---

## 3. Inventory — 425 tests in 78 files

### 3.1 Unit (370)

| Area | Tests | What they pin down |
|---|---|---|
| `core` | 29 | config parsing/validation; `FakeClock`/`SystemClock`; token-bucket rate limiter + retry/backoff (hypothesis); local blob store incl. path-traversal rejection; S3 vs local backend selection; engine factory |
| `ingest/edgar` | 77 | accession parse/format (hypothesis round-trip); EDGAR client etiquette (UA, 8 req/s, 429/5xx retry — respx); daily-index `.idx` parsing; submissions → company profile + filings (null `exchanges`); companyfacts → facts through the 43-concept allowlist; facts/ticker/submissions sync services with in-memory sinks (primary-ticker rule, `active_from` reuse, retire-on-drop); universe filter; bulk companyfacts target selection; bulk submissions member parsing; `bulk_archive` retry/backoff over `MockTransport`; streaming `zipindex` incl. a 65,541-entry zip64 archive; handler → snapshot enqueue |
| `filings` | 13 | 10-K section extraction (Item 1/1A/7 boundaries, TOC/cross-reference filtering); passage chunking |
| `xbrl` | 17 | the metrics engine: quarterly isolation, first-reported dedup, annual window, stock instants, concept fallback + notes, cross-period coalescing, YTD → Q4 derivation, non-additive EPS, FCF = OCF − capex, derived gross profit (untagged / tagged wins / operands resolved), missing denominators, period limits; allowlist |
| `market` | 15 | valuation multiples (positive-denominator rule, EV, yields); Yahoo adapter parsing + errors (respx); price fan-out handler + priority; bulk price CLI batching |
| `enrich` | 15 | budget breaker (daily cap, ledger); section locator/extractor over fakes; embedding builder + OpenRouter embedder (respx) |
| `scoring` | 34 | percentile ranking (ties, direction, renormalized weights, coverage); rule validation/evaluation (ops, `None` never matches, `gics_sector`); cohort resolver precedence (rule > ad-hoc > named) with a fake source; snapshot maths (fundamentals split, factors at a price, cik grouping, `snapshot_row`); snapshot job handlers |
| `universe` | 42 | the SIC → GICS crosswalk (representative code per sector, overrides beat major groups, unmapped codes) |
| `thesis` | 41 | reverse-DCF (solver, bracket, sensitivity grid, revenue mode, degenerate inputs); implied-expectations assembly (three bases, cyclical flag, buyback CAGR + split guard, fiscal-year rows); check vocabulary (parse / reject / schema); engine (hysteresis both polarities, roll-up table, near-tripwires, end-to-end report with diff and anchor, unresolvable → unknown, spec validation); job handler |
| `jobs` | 16 | queue semantics with fakes; worker registry; scheduler wiring from settings (interval + cron: price sync, snapshot revalue, thesis evaluate; enable/disable) |
| `mcp` | 59 | OAuth flow (18), auth + response shaping, ops tools (`request_*` two-step previews), implied tool shaping (14), cohort/thesis/sector helpers, server registration roster (36 tools), prompts, instructions pinned per feature |
| top-level / scripts | 12 | app factory; in-memory fakes behave like repositories; the EDGAR explorer spike (hypothesis) |

### 3.2 Integration (55)

| File | Tests | Path exercised |
|---|---|---|
| `test_migrations` | 1 | Alembic upgrade from empty to head on a fresh database |
| `test_repositories` | 6 | company/ticker/filing/fact repositories against Postgres (upsert semantics, COALESCE profile merge, point-in-time fact reads) |
| `test_jobs` | 7 | `FOR UPDATE SKIP LOCKED` claim/complete/fail, priorities, reaper of stale locks |
| `test_edgar_sync`, `test_facts_sync`, `test_daily_and_fetch`, `test_backfill` | 6 | the EDGAR sync services end-to-end with a fake client and real repositories; daily index → fetch → blob |
| `test_bulk_companyfacts`, `test_bulk_submissions` | 5 | the two bulk archive ingests over synthetic zips (targets only, `since`, malformed member skipped, SIC landed) |
| `test_s3_blobstore` | 2 | S3-compatible store against MinIO |
| `test_market` | 4 | price upsert, latest close, point-in-time close |
| `test_enrich`, `test_search` | 8 | extraction rows + embeddings persisted; pgvector semantic search over enrichment and filing chunks |
| `test_screen`, `test_cohorts`, `test_snapshots` | 4 | snapshot build → screen reads it (`basis`) → revalue moves multiples → refresh moves fundamentals → `as_of` bounded; saved static/rule cohorts; GICS rule; `list_sectors` |
| `test_implied`, `test_thesis_resolver`, `test_thesis` | 3 | implied expectations over seeded facts + price; every check type through the SQL resolver (risk → unknown without an embedder; point-in-time before a filing); thesis tools end-to-end (preview saves nothing, save records baseline, price move → broken + diff, backtest not recorded, replace keeps history, two-step delete) |
| `test_mcp_tools` | 9 | the read/ops tools over seeded data, `get_ingest_status` |

---

## 4. Coverage (statement coverage of `wda/`, 5,492 statements)

| Package | Statements | Unit only | Unit + integration |
|---|---|---|---|
| `xbrl` (metrics engine, repository) | 275 | 84% | **94%** |
| `thesis` | 799 | 72% | **92%** |
| `scoring` | 572 | 66% | **91%** |
| `cohorts` | 48 | 42% | 90% |
| `ingest` | 1,129 | 78% | 87% |
| `filings` | 141 | 69% | 87% |
| `mcp` | 919 | 55% | 86% |
| `universe` | 94 | 52% | 85% |
| `core` | 420 | 78% | 81% |
| `jobs` | 329 | 60% | 79% |
| `market` | 250 | 70% | 78% |
| `enrich` | 464 | 62% | 72% |
| `wda/app.py` | 52 | 63% | 63% |
| **Total** | **5,492** | **68%** | **85%** |

The gap between the two columns is the point of the strategy: the pure cores are
covered by unit tests; the shells are covered by integration tests. The remaining 15%
is concentrated, not spread:

| Module | Combined | Why it is uncovered | Matters? |
|---|---|---|---|
| `*/ports.py` (filings, universe, xbrl) | 0% | `Protocol` definitions — no executable lines worth counting | no |
| `scoring/snapshot_cli.py`, `ingest/edgar/bulk_*_cli.py`, `thesis/calibrate_cli.py`, `market/bulk_prices_cli.py` | 0-64% | one-off CLIs: env parsing + a call into a tested service; exercised by hand on the worker | low — the service they call is tested |
| `enrich/handlers.py` | 18% | the LLM job handlers (extract / embed / index) — need a fake LLM + fake embedder wired through the container; only the budget path is unit-tested | **medium** — the path that spends money and the one that silently failed twice on 2026-09-16 (statement timeouts) |
| `market/service.py` | 50% | price sync service beyond the happy path | low-medium |
| `ingest/edgar/handlers.py`, `market/handlers.py` | 58-76% | job handlers that compose services; the daily-index handler's "today" logic is untested — and wrong (§5) | **medium** |
| `jobs/scheduler.py`, `jobs/worker.py` | 68% | the run loops (APScheduler process, worker poll loop) — tested only via wiring | medium |
| `mcp/server.py` | 68% | the tool *bodies* are one-line wrappers; registration and instructions are tested, the FastMCP transport is not | low |
| `core/container.py`, `wda/app.py` | 63-74% | composition-root branches (S3, LLM client, embedder) not all instantiated in tests | low |
| `thesis/resolver.py` `risk_present` | 72% | the semantic branch needs an embedder; only the "no embedder → unknown" path runs | medium |

---

## 5. Risk register — what the tests do *not* prove

Ordered by how much it matters, with the evidence from production.

| # | Risk | Evidence | Mitigation in place | Gap |
|---|---|---|---|---|
| 1 | **Third-party behaviour is mocked.** EDGAR, Yahoo, Anthropic, OpenRouter are `respx`/fakes; their real error modes are not exercised. | SEC 403 on not-yet-published daily-index files (failing every run since 09-12); 429s on bulk archives from fly's shared IP; Yahoo raw-vs-adjusted close semantics | retry/backoff in the client and bulk helper (tested against mocked 429s) | no opt-in *contract* test that hits the live endpoints with a recorded-vs-live comparison; the daily-index "today" bug was invisible to every test |
| 2 | **Performance at production volume is untested.** Tests seed a handful of rows. | the whole-universe screen wedged the web app at 5,145 names; exact `count(*)` over 5.8M facts timed out; 990k-member zip thrashed the worker; a whole filing's vectors in one statement timed out five times (AECOM, 2026-09-17); the nightly 1,100-row-per-ticker price rewrite took 2 h | snapshots; estimates; streaming zip reader; batched resumable vector writes; 35-day price window; **and the database tier (Supabase Large, 2026-09-17) — the floor moved from ~100 IOPS to an in-memory working set** | no seeded-at-scale benchmark, no latency budget assertion, no I/O-contention scenario |
| 3 | **LLM extraction quality has no oracle.** Extraction tests check shape and persistence, not correctness of the risk themes / products / statements. | `is_new` diffs and theme dedup are prompt-driven; nothing measures precision/recall | typed schemas, dedup, budget ledger | no golden set (a few 10-Ks with human-labelled expected rows) and no drift check across model versions |
| 4 | **The deployed system is not smoke-tested.** CI proves the code; nothing proves the running service after deploy. | web machine auto-stopped when idle → "MCP is down"; migration/health timing between deploy and worker roll | fly health check on `/health`; release step runs migrations | no post-deploy probe of an MCP tool round-trip (e.g. `list_cohorts`) and of `get_ingest_status.snapshots` freshness |
| 5 | **Job handlers with side effects are thinly tested.** Especially `enrich.*`. | `enrich.embed` / `index_filing` jobs failed on DB statement timeouts and stayed failed silently (2026-09-16/17) | batched resumable vector writes (`vectors.py`, unit-tested over fakes); retry backoff 2→32 min; `get_ingest_status.recent_failures` and `get_pipeline_status(cik)` surface every failure with its error; the worker and scheduler survive a database restart (unit-tested) | the extract handler itself is still untested through fakes; no push alert on failed jobs (pull only) |
| 6 | **Data-semantics choices are tested as implemented, not validated against an external truth.** | first-reported restatement rule; fiscal-year weighted-average shares unadjusted for splits (NVDA 10:1 inside the price window); derived gross profit; EV = LTD − cash only | each is documented (`fundamentals-and-valuation.md`) and unit-tested for the chosen behaviour | no reconciliation test against an independent source (e.g. a handful of hand-checked company-years) |
| 7 | **Point-in-time is enforced in code paths, not proven globally.** | mixed-date evaluation would be a silent bug (thesis §6) | `as_of` threaded through facts, prices, screen, thesis; integration tests assert "before the filing" views | no property-style test that *every* dated read ignores rows filed after `as_of` across all repositories |
| 8 | **Migrations are tested forward only.** | — | `test_migrations` upgrades from empty to head | no downgrade test; no test of upgrading a populated database |
| 9 | **The scheduler singleton and the worker loop run untested.** | — | wiring tests (which crons exist, with what schedule) | the advisory-lock singleton, the poll loop, graceful shutdown |
| 10 | **Nondeterminism.** | none observed | hypothesis for the property-based tests; seeded fixtures elsewhere | none — the suite is deterministic today |

---

## 6. Recommendations, in order of value per hour

1. **Post-deploy smoke test** in the deploy job: hit `/health`, then one MCP tool
   (`list_cohorts`) and `get_ingest_status`, fail the job if either errors. Catches the
   whole class of "green CI, dead service". Hours.
2. **Extract-handler tests through fakes** (fake LLM client, real in-memory
   repositories) — the embed/index paths and the retry policy landed 2026-09-17 (#89,
   #91); the extraction path is the remaining gap in risk 5. Hours.
3. **Fix and test the daily-index sync** (previous business day + retry + backfill of
   missed days); add an **opt-in live contract test** (`-m live`, nightly, not on PR) for
   the EDGAR daily index, submissions and companyfacts endpoints and the Yahoo chart
   payload. Closes the visible part of risk 1. Half a day plus the fix.
4. **A coverage floor in CI** — fail under 80% combined — so the number cannot drift
   down unnoticed; report per-package. An hour.
5. **A scale benchmark** (opt-in): seed 5,000 synthetic companies' snapshots and prices,
   assert `screen()` < 1 s and the streaming build < N s; run nightly. Closes risk 2's
   regression exposure. Half a day.
6. **A golden set for enrichment**: three 10-Ks with human-labelled expected themes /
   products / statements; a scored comparison run on demand and on model change. Closes
   risk 3's blind spot. A day, mostly labelling.
7. **Point-in-time property test**: for each repository with an `as_of`, seed rows on
   both sides of a date and assert none from after it appear, generated with hypothesis.
   Closes risk 7. Hours.

---

## 7. How to keep this document true

- When a package's coverage moves more than five points, or a new package appears,
  re-run the two coverage commands from the header and update §4.
- When a production incident produces a regression test, add a line to §1 principle 4
  and, if it exposed a new class of risk, to §5.
- When one of §6's items ships, delete it here and record it in `progress.md`.
