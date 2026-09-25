# WorldDominationApp — Micro-Architecture

**Version:** 1.0 (2026-09-16)
**Audience:** whoever is about to change a module. [architecture.md](architecture.md) is
the system level; this is the module level — what each package contains, what its key
classes and functions do, and the call flow behind every user-visible feature.

---

## 1. Package by package

### `wda/core` — infrastructure, layer-agnostic

| Module | Contents |
|---|---|
| `config.py` | `Settings` (pydantic-settings, `WDA_*`), `get_settings()` (cached) |
| `db.py` | `Base`, `create_engine` (asyncpg, pre-ping, `statement_cache_size=0`), `create_sessionmaker`, `session_scope` |
| `http.py` | `RateLimiter` port, `TokenBucketLimiter`, `NoopLimiter`, `RateLimitedClient` (UA, gzip, retry 429/5xx with backoff) |
| `blobstore.py` | `BlobStore` port, `LocalBlobStore`, `S3BlobStore`, `edgar_document_key()` |
| `errors.py` | `WdaError` → `ConfigError`, `NotFoundError`, `ValidationError`, `IngestError`, `RateLimitError`, `ParseError`, `ProviderError`, `BudgetExceededError` |
| `clock.py` | `Clock` port, `SystemClock` (tests inject `FakeClock`) |
| `logging.py` | structlog configuration; JSON in prod |
| `container.py` | `Container`: the composition root. Builds every service from settings and exposes accessors (`screen_service()`, `cohort_resolver()`, `snapshot_builder()`, `implied_expectations()`, `check_resolver()`, `thesis_engine()`, `thesis_service()`, `job_queue()`, `job_context()`, `screenable_ciks()` …). The only module that imports across sibling layers. |

### `wda/universe` — companies and tickers

`models.py` (`Company`, `Ticker`); `repository.py` — `SqlCompanyRepository` (COALESCE-ing
`upsert` so a partial source never blanks a fuller one; `search`; `sectors_for` → (SIC,
description); `ciks_missing_sic`; `sector_counts`) and `SqlTickerRepository` (`upsert`,
`latest_active_from`, `close`, `list_for_cik`, `primaries`); `gics.py` — `gics_for_sic`
(4-digit overrides, then 2-digit major group; pure).

### `wda/filings` — filings, documents, narrative sections

`models.py` (`Filing`, `FilingDocument`); `repository.py`; `sections.py` — `html_to_text`,
`extract_item` / `extract_item_1a` (Items 1, 1A, 7 out of 10-K HTML with TOC and
cross-reference filtering); `chunking.py` — `chunk_text` for the passage index.

### `wda/xbrl` — facts and the canonical metrics engine

`allowlist.py` (`is_allowed`, 43 concepts); `models.py` (`XbrlFact`, partitioned by
`fy`); `repository.py` — `SqlFactRepository` (`add` append-only, `facts_for_concepts(cik,
concepts, as_of)`, `ciks_with_facts`, `approx_count` from planner stats); `metrics.py` —
`CATALOG` (`MetricDef`: concepts in preference order, kind flow|stock, unit, additive,
`derived_from`), `RATIOS` (`RatioDef`), `resolve(keys)` (expands ratios and derivation
operands), `concepts_for`, `build_series(cik, facts, keys, freq, periods)` →
`CompanySeries(rows, concepts_used, notes)`. Rules inside `build_series`: unit filter →
first-reported dedup per period → annual (350–380 d) / quarterly (80–100 d, Q4 = FY −
9M YTD for additive flows) selection → stock instants → derived fallbacks (gross profit)
→ ratios (FCF = OCF − capex as a difference). Pure; the most-tested module.

### `wda/ingest/edgar` — EDGAR ingestion

| Module | Contents |
|---|---|
| `client.py` | `EdgarClient` over `RateLimitedClient`: submissions, companyfacts, daily index, filing index, documents |
| `parsers/` | `Accession` (parse/format), `parse_master_idx`, `company_from_submissions` / `submissions_to_filings`, `facts_from_doc` (+ `concept_frequency`, `needs_mapping`), `parse_company_tickers_exchange` |
| `schemas.py` | Pydantic models for every EDGAR payload (`SubmissionsDoc` drops null exchanges) and the records the sinks accept (`CompanyRecord`, `FilingRecord`, `FactRecord`, `TickerRecord`) |
| `ports.py` | sinks: `CompanySink`, `TickerSink`, `FilingSink`, `FilingStatusSink`, `DocumentSink`, `FactSink` |
| `services/` | `TickerSyncService` (`primary_ticker` rule, `active_from` reuse, retire dropped), `SubmissionsSyncService`, `FactsSyncService` (allowlist, watermark), `DailyIndexSyncService`, `FilingFetchService`, `BulkBackfillService`, `BulkCompanyFactsService` (zip, since-year filter, per-company try/except), `BulkSubmissionsService` (profiles only, via `zipindex`) |
| `bulk_archive.py` | `download_archive` (429/5xx retry, `Retry-After`), `ensure_archive` |
| `zipindex.py` | streaming central-directory reader (zip64): `iter_members`, `find_members`, `read_member` |
| `universe_filter.py` | which filers belong in the universe |
| `handlers.py` | `edgar.sync_tickers`, `edgar.sync_submissions`, `edgar.sync_companyfacts` (→ enqueues `scoring.snapshot`), `edgar.sync_daily_index`, `edgar.fetch_filing` |
| CLIs | `bulk_companyfacts_cli`, `bulk_submissions_cli` (env-driven; `WDA_BULK_*`) |

### `wda/market` — prices and valuation

`ports.py` (`MarketDataProvider`, `DailyClose`, `MarketDataError`); `adapters/yahoo.py`
(`YahooProvider`: chart API, raw close — market cap correctness — `range=5y` or
`period1` since); `models.py` (`PriceDaily`, UNIQUE(ticker, date)); `repository.py`
(`upsert`, `latest_close(ticker, as_of)`, `latest_close_dated`, `latest_closes()` set-based,
`recent`); `service.py` (`PriceSyncService`, 1,100-day initial history, watermark per
ticker); `valuation.py` (`VALUATIONS`, `compute_valuation` — strictly-positive
denominators, EV = mcap + LTD − cash); `handlers.py` (`market.sync_prices_all` fan-out at
priority −10, `market.sync_prices`); `bulk_prices_cli`.

### `wda/enrich` — LLM extraction and semantic search

`extractors.py` — `EXTRACTORS: dict[target, TargetSpec]` (section to feed, schema, prompt,
ORM row class, `to_rows`, `to_dict`, `to_embed`, `needs_prior`) for `risk_themes`
(cross-filing `is_new`), `products`, `forward_statements`, `events`; `service.py` —
`SectionLocator` (find the 10-K item text from the blob) and `SectionExtractor` (call the
LLM, validate, replace rows, log `token_usage`); `budget.py` — `BudgetGuard` (daily USD
cap; `BudgetExceededError`), `cost_usd`; `embed_service.py` — `EmbeddingBuilder` (rows →
vectors) and `SectionIndexer` (10-K sections → chunks → vectors); `embeddings.py`
(`VectorEmbedder` port); `adapters/` — `AnthropicLlmClient`, `OpenRouterEmbedder`;
`repository.py` — `SqlEnrichmentRepository` (rows per target/accession, `distinct_ciks`,
`prior_risk_context`), `SqlTokenUsageRepository`, `SqlEmbeddingRepository` and
`SqlChunkRepository` (pgvector cosine search); `handlers.py` — `enrich.extract`,
`enrich.embed`, `enrich.index_filing`.

### `wda/cohorts` — saved cohorts

`models.py` (`Cohort` slug/label/kind static|rule/rule JSONB, `CohortMember`);
`repository.py` (`SqlCohortRepository`: upsert, get, list_all, delete → `CohortRecord`).

### `wda/scoring` — L5

| Module | Contents |
|---|---|
| `factors.py` | `FACTORS` (12 `FactorDef`: bucket, direction, default weight), `default_weights` |
| `scorer.py` | `score_cohort(values, weights)` → `ScoredCompany` (composite, coverage, per-factor `FactorCell` percentile); ties average; weights renormalized over present factors |
| `rules.py` | `STRING_FIELDS` (`sector`, `gics_sector`), operator sets, `rule_fields`, `validate_rule`, `compare`, `evaluate` (a `None` field never matches) |
| `cohorts.py` | `BUILTIN_COHORTS`, `CohortSources` port, `CohortResolver.resolve(cohort|ciks|tickers|rule)` (precedence rule > ad-hoc > named; rule cohorts re-select every time), `SqlCohortSources` (`screenable` = has a snapshot; `factor_rows` adds `sector` + `gics_sector`) |
| `snapshots.py` | pure: `METRIC_KEYS`, `concepts_needed`, `FactRow`, `fiscal_rows`, `fundamentals_from_rows` → `Fundamentals(quality, inputs)`, `factors_from(fundamentals, price)`, `group_by_cik` |
| `models.py` / `repository.py` | `FactorSnapshot`; `SqlFactorSnapshotRepository` (`upsert`, `revalue` executemany, `factors_for`, `ciks`, `valuation_inputs`, `stats`) |
| `snapshot_builder.py` | `snapshot_row`; `SnapshotBuilder.build_all` (one cik-ordered stream over recent partitions, 5k-row chunks, 500-row upserts), `refresh_cik`, `revalue_all` |
| `service.py` | `ScreenService.factor_values(ciks, as_of)` (snapshot; on-demand fill for ≤ 50 missing; `as_of` requires ≤ 50 explicit) and `screen()` |
| `handlers.py` / `snapshot_cli.py` | `scoring.snapshot`, `scoring.revalue_all`; the initial build / rebuild |

### `wda/thesis` — forward analysis and the thesis engine

| Module | Contents |
|---|---|
| `reverse_dcf.py` | `DcfAssumptions`, `dcf_value_per_share`, `implied_fcf_growth` (bisection, bracket −90%..+100%/yr), `implied_revenue_growth`, `sensitivity_grid` (WACC 8/10/12 × terminal 2/3), `ImpliedStatus` |
| `implied.py` | `assemble_inputs` (three bases: latest / mid-cycle median margin / peak; `cyclical` flag; buyback CAGR with the split guard), `fiscal_year_rows` (filter before cap), `ImpliedExpectationsService.for_cik(cik, as_of, assumptions, target_fcf_margin)` |
| `checks.py` | the check vocabulary (discriminated union: `financial_metric`, `valuation_metric`, `factor_percentile`, `risk_present`, `implied_expectations`), `parse_check`, `check_schema`, `threshold_text` |
| `schemas.py` | `ThesisSpec` / `ConditionSpec` / `Assumption` (enums: stance, conviction, polarity, severity, provenance) and the report (`ConditionResult`, `ThesisEvaluation`) |
| `engine.py` | pure judging: `judge_numeric` (hysteresis in periods, both polarities), `judge_boolean`, `rollup`, `near_tripwire`, `changes`; `ThesisEngine.evaluate(spec, as_of, previous)`; `compute_anchor`; `CheckResolver` port |
| `resolver.py` | `SqlCheckResolver` — each check type bound to its owning service, point-in-time |
| `models.py` / `repository.py` | four tables; `SqlThesisRepository` (spec round-trip, evaluation history, status roll-forward) |
| `service.py` | `ThesisService`: `preview`, `save`, `get`, `latest`, `history`, `list_all`, `delete`, `evaluate` (current recorded + diffed; `as_of` backtest not recorded), `evaluate_all` |
| `handlers.py` / `calibrate_cli.py` | `thesis.evaluate_all`; the adversarial-panel calibration printout |

### `wda/mcp` — the tool server

`server.py` — `create_mcp(container)`: `SERVER_INSTRUCTIONS`, 36 `@mcp.tool()`
registrations (thin wrappers), 3 prompts; `tools.py` — every tool's implementation over
services (payload shaping, two-step previews, `_normalize_slug`, `summarize_sources`,
`shape_sectors`, `shape_implied` …); `shaping.py` — `bound_rows`, `paginated`,
`DEFAULT_MAX_ROWS`; `auth.py` — bearer middleware; `oauth.py` — the fixed-client OAuth
2.1 provider for claude.ai connectors.

### `wda/jobs` — queue, worker, scheduler

`models.py` (`Job`); `queue.py` — `JobQueue.enqueue / claim / complete / fail / reap_stale /
stats`, `backoff_delay`; `registry.py` — `HandlerRegistry`, `@job_handler`; `context.py` —
`JobContext(sessionmaker, clock, container)`; `worker.py` — `Worker` poll loop,
`build_worker` (imports handler modules to register them); `scheduler.py` — `Scheduler`
(`every`, `cron`, `every_call`), `SingleInstanceLock` (advisory lock), `configure_scheduler`
(the production cadence from settings).

---

## 2. Call flows

### 2.1 Adding a company (`request_sync` → … → rankable)

```
request_sync(cik, confirm)            MCP tool, two-step
  └─ JobQueue.enqueue edgar.sync_submissions {cik}, edgar.sync_companyfacts {cik}
worker: edgar.sync_submissions        SubmissionsSyncService.sync_cik
  └─ EdgarClient.get_submissions → company_from_submissions → SqlCompanyRepository.upsert
     submissions_to_filings → SqlFilingRepository.upsert (per filing)
worker: edgar.sync_companyfacts       FactsSyncService.sync_cik
  └─ EdgarClient.get_companyfacts → facts_from_doc(allowlist) → SqlFactRepository.add (append-only)
     watermark edgar.companyfacts:<cik>; then enqueue scoring.snapshot {cik} (priority -5)
worker: scoring.snapshot              SnapshotBuilder.refresh_cik
  └─ facts_for_concepts → fiscal_rows → fundamentals_from_rows → factors_from(price) → upsert
request_price_sync(cik)               market.sync_prices → YahooProvider → SqlPriceRepository.upsert
```
After this the company is in `screenable`, scored by `screen`, and readable by
`get_financials` / `get_valuation` / `implied_expectations`.

### 2.2 Enriching a 10-K (`request_fetch` → `request_enrichment` → `request_index`)

```
edgar.fetch_filing {accession}        FilingFetchService: EdgarClient → BlobStore.put(edgar/<cik>/<accn>/<doc>) → mark_fetched
enrich.extract {cik, target, accn}    SectionLocator (blob → Item 1|1A|7 text)
                                      → BudgetGuard.check → AnthropicLlmClient (typed schema per TargetSpec)
                                      → SqlEnrichmentRepository.replace(rows) → SqlTokenUsageRepository.record
                                      (risk_themes: prior filing's themes prepended → is_new)
enrich.embed {cik, target, accn}      EmbeddingBuilder: rows → to_embed text → OpenRouterEmbedder → enrichment_embeddings
enrich.index_filing {cik, accn}       SectionIndexer: sections → chunk_text → vectors → filing_chunks
search_enrichment / search_filings    embed(query) → cosine nearest neighbours (pgvector), filtered by target/cik/item
```

### 2.3 Screening (`screen` / `get_score` / rule cohorts)

```
screen(cohort|ciks|tickers|rule, weights, as_of, top_k)
  └─ CohortResolver.resolve            rule: base cohort → factor_rows (snapshot + sector/gics) → evaluate(rule)
                                       ad-hoc: cik_for_ticker; named: built-in or saved cohort
  └─ ScreenService.screen(ciks, as_of)
        factor_values: as_of None → SqlFactorSnapshotRepository.factors_for(ciks)  (+ on-demand fill ≤ 50 missing)
                       as_of set  → _on_demand (facts_for_concepts as_of → fiscal_rows → factors_from(latest_close as_of)), ≤ 50 ciks
        score_cohort(values, weights) → percentiles within the cohort → composite, coverage
  └─ _shape_scores (names) + basis {snapshot: price_date, valued_at | point_in_time}
```

### 2.4 The nightly chain (Mon–Fri, America/New_York)

```
18:00  market.sync_prices_all  → one market.sync_prices per company with facts (priority -10) → prices_daily
19:30  scoring.revalue_all     → valuation_inputs (all snapshots) + primaries() + latest_closes()  [3 set-based reads]
                                  → factors_from per company → revalue (executemany) → factor_snapshots.factors/price/valued_at
20:00  thesis.evaluate_all     → for each thesis: ThesisEngine.evaluate(spec, previous=latest) → record → status
every 15 min  reap_stale        → requeue jobs locked > 15 min
every N h     edgar.sync_tickers, edgar.sync_daily_index (the latter currently failing, see architecture §4)
```

### 2.5 Forward analysis (`implied_expectations`)

```
ImpliedExpectationsService.for_cik(cik, as_of, assumptions, target_fcf_margin)
  └─ facts_for_concepts(5 keys) → build_series(annual, 48 period-ends) → fiscal_year_rows (≤ 12 FY rows)
  └─ primary ticker → latest_close(as_of)
  └─ assemble_inputs → DcfInputs (base_fcf, normalized_fcf [revenue × median margin], peak_fcf, cyclical, buyback_yield)
  └─ implied_fcf_growth on each base; implied_revenue_growth at the target margin; sensitivity_grid
tool: shape_implied → reading, band first, three bases, per_share translation, revenue_mode, provenance, L4 narrative
```

### 2.6 A thesis, from words to a monitored object

```
get_thesis_schema                     check vocabulary + shapes (the model composes conditions from the user's words)
create_thesis(..., confirm=false)     ThesisService.preview: ThesisSpec.model_validate → compute_anchor (reverse-DCF now)
                                        → ThesisEngine.evaluate: per condition → SqlCheckResolver.<type>(…, as_of)
                                            financial_metric → build_series(window, periods) values, most recent first
                                            valuation_metric → fiscal_rows + latest_close → compute_valuation[metric]
                                            factor_percentile → CohortResolver + ScreenService.screen → percentile
                                            risk_present     → embed(query) → enrichment_embeddings search (cik, target) → is_new / filed_at filters
                                            implied_expectations → ImpliedExpectationsService → result_latest | result_revenue
                                          → judge_numeric / judge_boolean → rollup → near_tripwires → anchor pinned vs now
create_thesis(..., confirm=true)      preview again → SqlThesisRepository.upsert + record_evaluation (baseline)
evaluate_thesis(name[, as_of])        evaluate vs latest recorded → changes → record (as_of None) | return only (as_of)
thesis.evaluate_all (20:00 ET)        the same, for every thesis; status rolls forward
```

### 2.7 Bulk paths (one-off CLIs on the worker)

- `bulk_companyfacts_cli`: `ensure_archive(companyfacts.zip)` → `BulkCompanyFactsService.ingest_zip(targets, since)` via `zipfile` (18k members) → `SqlFactRepository.add` per company. `WDA_BULK_ONLY_MISSING` skips companies with facts.
- `bulk_submissions_cli`: `ensure_archive(submissions.zip)` → `BulkSubmissionsService.ingest_zip` via `zipindex.find_members` (990k members, one directory pass) → profile-only `upsert`.
- `snapshot_cli`: `SnapshotBuilder.build_all(since_fy)` — one streaming SQL pass, grouped per cik; ~90 s for the universe outside the price window.

---

## 3. Cross-cutting rules that every module obeys

- **Point-in-time**: every fact read takes `as_of` and filters `filed_at <= as_of`
  before selection; prices filter `date <= as_of`; the primary ticker is resolved for
  the date where it matters (thesis/forward). Never mix a dated fact read with an
  undated price read.
- **Append-only facts**; derived layers are functions of the log and rebuildable.
- **Honest absence**: a missing factor is absent (never zero), a non-positive
  denominator yields no multiple, an unresolvable check is `unknown`, a partial
  horizon is flagged. Nothing silently passes.
- **Bounded reads**: tool responses are shaped; universe-wide on-demand computation is
  refused (`ON_DEMAND_MAX`); heavy reads are set-based (`DISTINCT ON`, streaming) or
  served from the snapshot.
- **Two-step writes** for anything that spends or persists through MCP.
- **Layer contract** enforced by import-linter; siblings bridged only in `Container`.
