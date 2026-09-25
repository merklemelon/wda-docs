# EDGAR Ingestion — Research and Strategy

**Version:** 1.0-dev
**Scope:** how we discover, fetch, store and refresh SEC filings and XBRL facts.
Owning package: `wda/ingest/edgar/`. Raw storage: `wda/filings`, `wda/xbrl/facts`.

---

## 1. EDGAR surfaces we will use

Two hostnames, different purposes, same access rules:

| Host | What lives there | Notes |
|---|---|---|
| `https://www.sec.gov` | Filing archives, indexes, bulk zips, ticker lookup | Static-ish files |
| `https://data.sec.gov` | JSON APIs: submissions, XBRL companyfacts / companyconcept / frames | Real-time-ish, gzip |

### 1.1 Identifiers

- **CIK** — integer, canonical company id. Zero-pad to 10 digits in URLs (`CIK0000320193`).
- **Accession number** — `0001193125-15-118890` = filer-agent CIK (10) + year (2) + sequence (6).
  Unique per submission. Directory form strips dashes: `000119312515118890`.
- **Ticker** — mutable; never a primary key. Map via
  `https://www.sec.gov/files/company_tickers.json` (cik, ticker, title) and
  `https://www.sec.gov/files/company_tickers_exchange.json` (adds exchange).

### 1.2 Discovery endpoints

| Purpose | URL | Cadence we poll |
|---|---|---|
| Ticker → CIK universe | `www.sec.gov/files/company_tickers_exchange.json` | daily |
| Per-company filing history | `data.sec.gov/submissions/CIK##########.json` (+ `filings.files[]` for older pages) | backfill + reconcile |
| All filers, bulk | `www.sec.gov/Archives/edgar/daily-index/bulkdata/submissions.zip` | initial backfill |
| What was filed today | `www.sec.gov/Archives/edgar/daily-index/YYYY/QTRn/form.YYYYMMDD.idx` (also `master.`, `crawler.`, `xbrl.`) | 3–4×/day |
| Quarter index | `www.sec.gov/Archives/edgar/full-index/YYYY/QTRn/form.idx` | gap repair |

The daily index files are pipe/space-delimited text with a header block; the
`master` variant is `CIK|Company Name|Form Type|Date Filed|Filename` and is the
easiest to parse. Files appear for business days only.

### 1.3 Document endpoints

Filing folder: `www.sec.gov/Archives/edgar/data/<cik>/<accession_nodash>/`
- `index.json` — machine-readable listing of the folder (`directory.item[]`)
- `<accession>-index.htm` — human index
- `<accession>.txt` — full submission text file (all documents concatenated; large)
- Primary document filename comes from the submissions JSON (`primaryDocument`)
- Inline XBRL filings also have `Financial_Report.xlsx` and `R*.htm` rendered pages;
  ignore both.

### 1.4 XBRL endpoints

| Endpoint | Returns | Use |
|---|---|---|
| `data.sec.gov/api/xbrl/companyfacts/CIK##########.json` | every fact for one company, keyed taxonomy→concept→unit→[facts] | per-company backfill and reconcile |
| `data.sec.gov/api/xbrl/companyconcept/CIK##########/us-gaap/<Concept>.json` | one concept for one company | spot checks |
| `data.sec.gov/api/xbrl/frames/us-gaap/<Concept>/<unit>/CY2025Q4I.json` | one concept, one period, **all companies** | cross-sectional screens without touching every company |
| `www.sec.gov/Archives/edgar/daily-index/xbrl/companyfacts.zip` | all companyfacts JSON | **initial backfill** (one download vs 6k calls) |

Each fact carries: `start` (absent for instants), `end`, `val`, `accn`, `fy`, `fp`,
`form`, `filed`, and sometimes `frame`. That is exactly the point-in-time tuple we
need: `end` = `period_end`, `filed` = `filed_at`, `accn` = `accession_no`.

**Financial Statement Data Sets** (`www.sec.gov/dera/data/financial-statement-data-sets`)
are quarterly zips of `sub/num/pre/tag` tables. They are an alternative bulk
source with better presentation metadata. Evaluate in Phase 0 but default to
`companyfacts.zip` — same facts, simpler structure.

### 1.5 Access rules (enforced in `wda/core/http.py`)

- Max **10 requests/second** across all our machines combined; we run at 8.
- **Declared `User-Agent`** of the form `AppName contact@email` — undeclared
  clients are throttled or blocked.
- Send `Accept-Encoding: gzip, deflate`; `data.sec.gov` responses are gzipped.
- Back off on 403/429; SEC blocks for a "brief period" when exceeded.
- Bulk zips are republished nightly ~3:00 a.m. ET; the JSON APIs update in
  near real time. Filings are accepted 6 a.m.–10 p.m. ET business days.
- Note `www.sec.gov/Archives/...` paths and `data.sec.gov/...` paths are served
  by different infrastructure; keep one client but expect different latency.

---

## 2. Storage strategy

### 2.1 Principles
1. **Raw is immutable and complete.** We store the original JSON payloads
   (`companies.raw`, `filings.raw`) and the original documents (blob store).
   Parsers can always be re-run offline.
2. **Natural keys, upserts, watermarks.** Re-running any job is a no-op.
3. **Fetch documents lazily by form and tier.** Metadata for every filing in the
   universe; documents only for forms we analyze, and only for companies in
   the current tier (see §3).
4. **Facts are append-only.** Same period, different accession = new row.

### 2.2 What is stored where

| Data | Store | Key |
|---|---|---|
| Company metadata | `companies` (+ `raw` jsonb) | `cik` |
| Ticker history | `tickers` | (`ticker`, `cik`, `active_from`) |
| Filing metadata (all forms) | `filings` (+ `raw` jsonb from submissions) | `accession_no` |
| Document listing | `filing_documents` | (`accession_no`, `seq`) |
| Primary document bytes | blob store | `edgar/<cik10>/<accn_nodash>/<filename>` |
| Full submission `.txt` | blob store (only when primary doc is insufficient) | same folder |
| XBRL facts | `xbrl_facts` partitioned by `fy` | natural key incl. `accession_no` |
| Cursors | `ingest_watermarks` | `source` (`edgar.daily_index`, `edgar.tickers`, `edgar.companyfacts:<cik>`) |

### 2.3 Form types in scope

Phase 1: `10-K`, `10-K/A`, `10-Q`, `10-Q/A`, `8-K`, `8-K/A`, `DEF 14A`.
Metadata for `20-F`, `40-F`, `S-1`, `424B*` stored but documents not fetched.
Everything else (Forms 3/4/5, 13F, D, etc.) skipped at index-parse time by a
configurable allowlist (`settings.edgar_form_allowlist`).

### 2.4 Universe filter (from vision doc §3.6)

Start from `company_tickers_exchange.json` → CIKs with a ticker on
NYSE/Nasdaq/NYSE American. Enrich from submissions JSON (`sic`, `entityType`,
`stateOfIncorporation`). Exclude SIC 6726 and 6798-like fund/trust codes,
names matching `/\b(TRUST|FUND|ETF|ACQUISITION CORP)\b/` (SPAC/fund heuristic),
`entityType != "operating"`. Expect ~5–6k. Report additions/removals per run.

---

## 3. Ingestion pipeline (jobs)

```
edgar.sync_tickers        daily      company_tickers_exchange.json → companies/tickers
edgar.sync_submissions    per-cik    submissions JSON (all pages) → filings (metadata only)
edgar.sync_daily_index    3-4x/day   master.YYYYMMDD.idx → new filings (metadata) → enqueue fetch
edgar.fetch_filing        per-accn   index.json + primary doc → blob store, filing_documents
edgar.sync_companyfacts   per-cik    companyfacts JSON → xbrl_facts (allowlisted concepts)
edgar.backfill_companyfacts once     companyfacts.zip → xbrl_facts for whole universe
edgar.backfill_submissions  once     submissions.zip → filings for whole universe
edgar.reconcile           weekly     submissions JSON vs our filings; enqueue misses
```

Job dependencies: `sync_daily_index` only enqueues `fetch_filing` for CIKs in
`companies` and forms in the allowlist. `fetch_filing` for a 10-K/10-Q also
enqueues `sync_companyfacts` for that CIK (facts appear within ~a minute of
dissemination).

Tiering (vision doc §1): metadata + facts for all companies; documents fetched
for all companies but only for the in-scope forms; section splitting and LLM
work only for watchlist/candidate tiers (later phases).

### 3.1 Class map (`wda/ingest/edgar/`)

```
client.py         EdgarClient(http: RateLimitedClient)
                    get_company_tickers() -> list[TickerRow]
                    get_submissions(cik) -> SubmissionsDoc     (follows files[])
                    get_daily_index(date, kind="master") -> list[IndexRow]
                    get_filing_index(cik, accn) -> FilingIndex
                    get_document(cik, accn, filename) -> bytes
                    get_company_facts(cik) -> CompanyFactsDoc
schemas.py        Pydantic models for every payload above (raw shapes, validated)
parsers/
  idx.py          parse_master_idx(text) -> list[IndexRow]      (pure)
  submissions.py  submissions_to_filings(doc) -> list[FilingRecord]  (pure, flattens columnar arrays)
  companyfacts.py facts_from_doc(doc, allowlist) -> list[FactRecord]   (pure)
  accession.py    Accession value object: parse/format/dir form/validate
ports.py          FilingSink, FactSink, CompanySink, BlobStore, Watermarks (Protocols)
services/
  ticker_sync.py     TickerSyncService
  submissions_sync.py SubmissionsSyncService
  daily_index_sync.py DailyIndexSyncService
  filing_fetch.py    FilingFetchService
  facts_sync.py      FactsSyncService
  backfill.py        BulkBackfillService (streams zips, never loads whole zip into memory)
handlers.py       @job_handler bindings, thin
```

Parsers are pure functions over Pydantic input → dataclasses/Pydantic output
and are the most heavily unit-tested code in the package.

### 3.2 Failure handling
- Network/429/5xx: retried by client; job marked `retry` with backoff after
  client gives up.
- Parse failures: filing row gets `ingest_status='parse_error'` with message;
  raw payload retained; job succeeds (one bad filing does not poison a batch).
- Missing primary document: fall back to `.txt` full submission, flag it.
- Watermark advanced only after a successful batch.

---

## 4. Concept allowlist (initial)

Keep `xbrl_facts` an order of magnitude smaller by ingesting only concepts we
will map. Seed list in `wda/xbrl/allowlist.py` (~120 to start, grow to ~300):
Revenues, RevenueFromContractWithCustomerExcludingAssessedTax, SalesRevenueNet,
CostOfRevenue, CostOfGoodsAndServicesSold, GrossProfit, OperatingIncomeLoss,
NetIncomeLoss, EarningsPerShareBasic/Diluted, WeightedAverageNumberOf*Shares*,
ResearchAndDevelopmentExpense, SellingGeneralAndAdministrativeExpense,
DepreciationDepletionAndAmortization, InterestExpense, IncomeTaxExpenseBenefit,
Assets, AssetsCurrent, Liabilities, LiabilitiesCurrent, StockholdersEquity,
CashAndCashEquivalentsAtCarryingValue, LongTermDebt*, Inventory*,
AccountsReceivableNetCurrent, PropertyPlantAndEquipmentNet, Goodwill,
NetCashProvidedByUsedInOperating/Investing/FinancingActivities,
PaymentsToAcquirePropertyPlantAndEquipment, CommonStockSharesOutstanding,
plus `dei:EntityCommonStockSharesOutstanding`, `dei:EntityPublicFloat`.
Unknown-but-frequent concepts are counted into a `needs_mapping` report, not dropped silently.

---

## 5. Phase 0 exploration questions (answer with `scripts/explore_edgar.py`)

1. For 3–5 known companies (e.g. AAPL 320193, MSFT 789019, NVDA 1045810, a small-cap, a recent IPO): how many filings of each form in the last 5 years? Does `submissions` paginate?
2. What does the primary document look like for a 10-K — single `.htm`? Size? Inline XBRL?
3. How consistent is `fp`/`fy` in companyfacts? How many revenue-like concepts per company? How many restated period duplicates?
4. Does a daily `master.idx` line map cleanly to accession + primary doc, or do we always need `index.json`?
5. Is `edgartools` worth adopting for parsing? Compare its statement assembly on the same 5 companies against raw companyfacts (record findings in `docs/decisions.md`).
