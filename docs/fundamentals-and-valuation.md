# Fundamentals, Valuation and the Factor Snapshot — Data Lineage

**Status:** reference, describes the code as of 2026-09-15 (architecture 1.1-dev)
**Related:** `docs/architecture.md` §1.1 (layers) and §5.1 (persisted L5),
`docs/gics-sectors.md` (the `sector` / `gics_sector` rule fields),
`docs/forward-analysis.md` (the reverse-DCF consumes the same L3 rows).

This document answers two questions: *where does every number in a screen factor come
from, and what processing formed it?* and *where does the factor snapshot sit and how is
it kept current?* It is the map to consult before changing anything in
`wda/xbrl/metrics.py`, `wda/market/valuation.py` or `wda/scoring/`.

---

## 1. The pipeline at a glance

```
L0 ingest    EDGAR companyfacts JSON (per company, or the bulk zip)   Yahoo daily closes
                 |  allowlist of 43 concepts, pydantic-validated            |  MarketDataProvider (vendor-neutral)
L1 raw       xbrl_facts: one row per (cik, concept, unit, period, accession)  prices_daily (ticker, date, close, source)
                 |  append-only; period_end + filed_at on every row; partitioned by fiscal year
L2 normalize metric catalog: concept(s) -> canonical metric            primary-ticker selection
                 |  fallbacks, combining, kind (flow/stock), units
L3 derived   build_series: period selection, YTD->quarter, ratios      latest_close(ticker, as_of)
                 |  one row per fiscal year (or quarter) per company           |
L5 scoring   snapshot: quality + inputs (from L3 facts)  x  price (L3 market) -> factors -> percentiles
```

Two independent streams — **filings** and **prices** — that meet only at the top, in
`compute_valuation`.

---

## 2. Fundamentals (the filings stream)

### 2.1 L0 -> L1: what gets stored

- Source: a company's EDGAR *companyfacts* document (`wda/ingest/edgar/parsers/companyfacts.py`),
  fetched per company by `edgar.sync_companyfacts` or read from the bulk
  `companyfacts.zip` by `bulk_companyfacts_cli` (which keeps the last five fiscal years).
- Filter: the **allowlist of 43 concepts** in `wda/xbrl/allowlist.py` — revenue, costs,
  income lines, EPS, shares, cash-flow lines, balance-sheet items. Anything else in the
  filing is dropped at ingest.
- Storage: `xbrl_facts`, one row per fact with `period_start` / `period_end`, `filed_at`,
  `accession_no`, `fy` / `fp` / `form`, `unit`, `value` (`Numeric`, never float).
  Append-only; a restatement is a new row under its own accession (rule 4). Partitioned
  by `fy`; indexes on `cik`, `concept`, `accession_no` plus the natural-key unique index.
- Concepts a filer uses that no catalog entry covers are counted as `needs_mapping` by the
  sync report — the growth list for the catalog (mostly IFRS filers).

### 2.2 L2: the metric catalog (`wda/xbrl/metrics.py`)

A `MetricDef` per canonical metric lists the XBRL concepts that can supply it, in
preference order, with `kind` (`flow` | `stock`), `unit`, and `additive` (whether YTD
arithmetic is valid). This is where vendor idiosyncrasy is absorbed.

| Metric | Kind | Concepts (preference order) |
|---|---|---|
| revenue | flow | `RevenueFromContractWithCustomerExcludingAssessedTax` -> `Revenues` -> `RevenuesNetOfInterestExpense` |
| cost_of_revenue | flow | `CostOfGoodsAndServicesSold` -> ... |
| operating_income | flow | `OperatingIncomeLoss` |
| net_income | flow | `NetIncomeLoss` |
| eps_diluted | flow | `EarningsPerShareDiluted` (USD/shares) |
| shares_diluted | flow, non-additive | `WeightedAverageNumberOfDilutedSharesOutstanding` |
| operating_cash_flow | flow | `NetCashProvidedByUsedInOperatingActivities` (+ `...ContinuingOperations`, combined) |
| capex | flow | `PaymentsToAcquirePropertyPlantAndEquipment` -> `PaymentsToAcquireProductiveAssets` |
| cash | stock | `CashAndCashEquivalentsAtCarryingValue` |
| long_term_debt | stock | `LongTermDebtNoncurrent` -> `LongTermDebt` |
| equity, assets, current_assets, current_liabilities, inventory | stock | the us-gaap totals |

When a company switches concepts across years (Intel, NVIDIA) the engine combines them
and says so in `notes` (`"<metric>: combined concepts [...]"`, `"used fallback concept
..."`); `concepts_used` in every `get_financials` response records which concept won.

### 2.3 L3: `build_series` — facts to fiscal-year rows

Per company, per metric, in this order:

1. **Unit filter** — USD for money, `shares`, `USD/shares` for EPS.
2. **Restatement rule — `_first_reported`.** For each `(period_start, period_end)` the
   value from the **earliest filing** is kept. A later restatement of the *same* period
   does not move the number; a *new* fiscal year does. This is a deliberate point-in-time
   choice (the number the market saw first). A "latest filed wins" mode would be a small
   switch if restated views are ever wanted.
3. **Period selection.**
   *Flows*: annual = durations inside the `ANNUAL_DAYS` window (about one year);
   quarterly = about three months, with **Q4 derived as FY minus the nine-month YTD** for
   additive flows (noted as `"N quarter(s) derived from YTD differences"`).
   *Stocks*: instants at each period end. Shares are non-additive: no YTD arithmetic.
4. **Point-in-time.** `as_of` drops every fact with `filed_at > as_of` *before* steps
   1-3, so a dated query reproduces what was knowable on that date.
5. **Ratios** (`_apply_ratios`): gross / operating / net margin, R&D and SG&A intensity,
   **FCF = operating cash flow - capex**, FCF margin, current ratio, debt/equity.
   Percentages are x100 rounded to 1 decimal; multiples ("x") rounded to 2.

Output: `CompanySeries.rows`, most recent first, e.g.
`{"period_end": "2025-12-27", "revenue": 5.29e10, "operating_margin": -8.1, "fcf": -4.95e9, ...}`.

### 2.4 L5: the fundamentals half of the snapshot (`wda/scoring/snapshots.py`)

- `fiscal_rows` keeps only rows that carry revenue — a complete fiscal year.
- `fundamentals_from_rows` takes the latest row for the seven **quality** factors
  (fcf / operating / net / gross margin, current ratio, debt/equity) and computes
  **revenue growth = latest / prior - 1 (in %)** from the two most recent years; it also
  stores the eight raw **inputs** the multiples need (EPS, diluted shares, revenue, FCF,
  operating income, equity, cash, long-term debt) and `fiscal_period_end`.
- These become the `quality` and `inputs` JSONB columns of `factor_snapshots`.

---

## 3. Valuation (the prices stream)

### 3.1 L0 -> L1

`MarketDataProvider` (Yahoo behind the protocol; no vendor names in the schema) writes
`prices_daily` — daily closes per **ticker**, upserted by date. Nightly at 18:00 ET
`market.sync_prices_all` fans out one `market.sync_prices` job per company with facts
(about two minutes for ~5,150 tickers).

### 3.2 L2: which price is *the* company's price

The primary listing: hyphen-free -> shortest -> alphabetical (`primary_ticker`), with
`active_from` / `active_to` history in `tickers`. (This ordering is what fixed JPM being
priced off an exchange-traded note.)

### 3.3 L3: `compute_valuation` (`wda/market/valuation.py`)

Pure arithmetic on one close plus the latest-fiscal-year inputs.

| Multiple | Formula | Notes |
|---|---|---|
| market_cap | close x diluted shares (weighted average, latest FY) | |
| EV | market_cap + long_term_debt - cash | no short-term debt, leases, minorities, marketable securities |
| P/E | close / diluted EPS (latest FY) | trailing **fiscal year**, not TTM |
| P/S | market_cap / revenue | |
| P/B | market_cap / equity | |
| EV/EBIT | EV / operating income | |
| FCF yield | FCF / market_cap | a plain fraction: 0.064 = 6.4% |

Every denominator must be **strictly positive** or the multiple is `None`: a loss-making
P/E, a negative-equity P/B or a negative-EBIT EV/EBIT is treated as *absent*, not as
cheap. A missing factor neither helps nor hurts a score (weights renormalize), which is
why Intel screens "weak across margins" rather than accidentally "cheap".

### 3.4 L5: the valuation half of the snapshot

`factors_from(fundamentals, price)` merges `quality` with the five multiples -> the
`factors` JSONB column. The nightly revaluation redoes only this step, from the stored
`inputs` and the latest close; it never reads `xbrl_facts`.

---

## 4. The factor snapshot — where it sits and how it is updated

### 4.1 Placement

- **Layer:** L5, `wda/scoring`. Nothing below it imports it; the only lower-layer
  touchpoint is the facts-sync job *enqueuing* a refresh by job name (a string).
- **Table:** `factor_snapshots` (migration 0008, model `wda/scoring/models.py`), one row
  per company keyed by `cik`.

| Column | Holds | Changes when |
|---|---|---|
| `fiscal_period_end` | latest complete fiscal year the numbers came from | company files |
| `quality` (JSONB) | the 7 fundamentals-only factors + revenue growth | company files |
| `inputs` (JSONB) | the 8 latest-FY figures the multiples need | company files |
| `price`, `price_date` | the close the multiples were computed at | nightly |
| `factors` (JSONB) | the full factor dict the scorer reads (quality + multiples) | nightly, and on filing |
| `coverage` | how many of the 12 factors are present | with `factors` |
| `fundamentals_at`, `valued_at` | timestamps of each half's last refresh | respectively |

The split is the point: the expensive half (reading a company's facts — 10-30 s cold on
a ~100-IOPS disk) is done only when that company's facts change; the cheap half is redone
nightly from stored numbers and the latest close.

### 4.2 One implementation, three callers

`wda/scoring/snapshots.py` is pure: `fiscal_rows` -> `fundamentals_from_rows` ->
`factors_from`. The on-demand (point-in-time) screen, the streaming build and the nightly
revaluation all call these same functions, so they cannot drift from each other.

### 4.3 Update paths

**A. Initial build / full rebuild** — `SnapshotBuilder.build_all`, run by hand:
`python -m wda.scoring.snapshot_cli` on the worker (`WDA_SNAPSHOT_SINCE_FY`,
`WDA_SNAPSHOT_CIKS` to narrow). One SQL statement streams every fact for the needed
concepts filed under `fy >= current year - 3`, **ordered by cik**, through a server-side
cursor in 5,000-row chunks; facts are grouped per company as they stream (one company in
memory at a time) and written in batches of 500. Primary tickers and latest closes are
loaded once, set-based. 2026-09-15: 4,693 companies, 1.85M facts, ~90 s, 3,712 rows.
Companies whose last 10-K is older than the window get no snapshot. This is also the
"rebuild L5 from L1/L3 with one command" required by rule 3.

**B. Per-company refresh** — `refresh_cik`, the `scoring.snapshot` job. Enqueued by
`edgar.sync_companyfacts` after it commits new facts (priority -5: below fetch and
enrichment, above the price fan-out). Reads that one company's facts, recomputes **both
halves**, upserts the row. A company with facts but no revenue-bearing fiscal year writes
nothing and stays out of the screen.

**C. Nightly revaluation** — `revalue_all`, the `scoring.revalue_all` job. Cron
**19:30 America/New_York, Mon-Fri** (`snapshot_revalue_*` settings), 90 minutes after the
price fan-out. One set-based job: loads every row's `quality` + `inputs`, all primary
tickers, all latest closes (`SqlPriceRepository.latest_closes`, one `DISTINCT ON` pass),
recomputes `factors` in memory, writes back in batches with `price`, `price_date`,
`coverage`, `valued_at`.

### 4.4 Read paths (`wda/scoring/service.py`, `ScreenService.factor_values`)

The single gate for `screen`, `get_score` and rule cohorts (`factor_rows`):

- **`as_of` unset** -> `factors` from the snapshot for the requested ciks (or every row
  for the universe). Milliseconds (universe screen: 0.72 s end-to-end).
- **Explicit small cohort with names missing from the snapshot** (added since the build,
  refresh job not yet run) -> up to `ON_DEMAND_MAX` (50) of them are computed on demand
  and merged in, so an ad-hoc ticker comparison never silently drops a fresh name.
- **`as_of` set** -> always on demand from the facts as they stood on that date, and it
  *requires* an explicit cohort of at most 50; a universe-wide `as_of` screen is rejected
  with a message saying so (the guard against the request that wedged the web app).
- The `screenable` / `universe` built-in cohort is "every cik with a snapshot" (a 3.7k-row
  read), not `DISTINCT cik` over 5.8M facts. The nightly price fan-out still targets
  "every company with facts" on purpose — prices matter for `get_valuation` on banks too.
- Every `screen` response carries `basis`: `{"kind": "snapshot", "price_date",
  "valued_at"}` or `{"kind": "point_in_time", "as_of"}`.

### 4.5 Freshness, staleness, and how to see it

- `get_ingest_status` -> `snapshots: {count, fundamentals_at, valued_at, price_date}`.
- **Multiples**: at most one trading day old (`price_date`). If a price sync fails,
  `revalue_all` still runs with yesterday's closes and `price_date` does not advance —
  that is the tell.
- **Fundamentals**: refresh only when a facts sync runs for that company. Today that is
  triggered by `request_sync` (and the original ingestion); **there is no scheduled
  universe-wide facts refresh yet**, so a company that files a new 10-K keeps its previous
  year until something syncs it. The fix is a periodic facts sync (or one driven off the
  daily index when a 10-K/10-Q lands), which then flows into path B automatically.
- **Bulk backfills** (`bulk_companyfacts_cli`) do not enqueue refreshes; run the build
  CLI afterwards.
- **Leaving the screen**: rows are never deleted. A delisted company keeps its last
  fundamentals with a price that stops advancing; a rebuild overwrites but does not prune
  companies that fell below the window. A prune step is cheap to add if wanted.
- **Ticker changes**: `primaries()` reads the current primary listing at revaluation, so
  a re-ticker is picked up the next night.
- Worker log events: `snapshots.build.*`, `scoring.snapshot.done`, `snapshots.revalue.done`.

---

## 5. Caveats to keep in mind when reading a factor

- **Fiscal-year granularity.** The screen uses complete fiscal years, not TTM; quarterly
  data exists in L3 (`get_financials freq=quarterly`) but is not used for factors. A TTM
  mode (sum of the last four quarters) would sharpen the multiples and fix the next point.
- **Shares are the fiscal-year weighted average, unadjusted for splits** — after a split,
  market cap and P/S, P/B, FCF yield are wrong until the next 10-K (the NVIDIA 10:1
  hazard; same root as the buyback-yield split guard in `wda/thesis/implied.py`).
- **Balance-sheet items are instants at the fiscal year end**, so current ratio and
  debt/equity can be up to a year old.
- **EV is simplified**: long-term debt minus cash only.
- **Banks, insurers, REITs** mostly report no revenue concept the catalog knows -> no
  fiscal row -> no snapshot -> not ranked (981 companies skipped on 2026-09-15). They come
  in by extending the catalog (`InterestAndDividendIncomeOperating`, revenue variants).
- **Percentiles** (`wda/scoring/scorer.py`) are computed within the cohort over whatever
  factors a company has, weights renormalized. A company with one factor can therefore
  top the universe (coverage 1, current ratio only). A minimum-coverage floor for large
  cohorts is the queued fix.
- **Restatements** keep the first-reported value per period (§2.3).

---

## 6. Quick reference — files

| Concern | Where |
|---|---|
| ingest allowlist | `wda/xbrl/allowlist.py` |
| companyfacts parsing | `wda/ingest/edgar/parsers/companyfacts.py` |
| metric catalog, series builder, ratios | `wda/xbrl/metrics.py` |
| fact repository (point-in-time reads, estimates) | `wda/xbrl/repository.py` |
| valuation multiples | `wda/market/valuation.py` |
| prices, latest closes | `wda/market/repository.py` |
| primary ticker rule | `wda/ingest/edgar/services/ticker_sync.py` |
| snapshot math (pure) | `wda/scoring/snapshots.py` |
| snapshot build / refresh / revalue | `wda/scoring/snapshot_builder.py`, `snapshot_cli.py`, `handlers.py` |
| snapshot table + repository | `wda/scoring/models.py`, `wda/scoring/repository.py`, migration `0008` |
| screen read path, on-demand cap | `wda/scoring/service.py` |
| percentiles and composite | `wda/scoring/scorer.py`, `wda/scoring/factors.py` |
| schedule (19:30 ET revalue) | `wda/jobs/scheduler.py`, `wda/core/config.py` |
