# Forward Outcome — Point-in-Time Returns vs a Cohort

**Status:** proposal, 2026-09-15 (not built)
**Related:** [wda-flows.md](wda-flows.md) §3.4 and §8 (the objective function), [thesis.md](thesis.md)
(what an outcome attaches to), [fundamentals-and-valuation.md](fundamentals-and-valuation.md) §3
(the price stream).

---

## 1. What it is for

Every WDA tool answers "what is true / what was true on date X". None answers "what
happened *after* X". A thesis evaluated at a past date tells you whether its claims held
*then*; it cannot tell you whether the company then did what the thesis needed. That
second question is the **objective function** the flows proposal needs to score archetypes
— and the one thing a backtest cannot do without.

`get_forward_return` answers it, honestly and point-in-time: given a company, a decision
date and a horizon, what was the return from the first close *after* that date to the
close at the end of the horizon — and how did that compare with a cohort measured the same
way? It reports total and price return, an annualized figure, the worst drawdown on the
way, and the company's place in the cohort's distribution. It is a measurement, not a
signal; like everything else in the stack it carries its provenance and its gaps.

---

## 2. Definitions

**Decision date and entry.** `as_of` is the date on which a decision could have been
made with the information WDA would have had (facts filed by then, closes up to and
including that day). The position is taken at the **first close strictly after
`as_of`** — you cannot buy at a close you needed to see first. If no close exists within
the next 10 calendar days (a halt, a listing gap), the read is `no_entry`.

**Exit.** The **last close on or before `as_of + horizon_months`**. If that date is in
the future, the exit is the latest close on record and the result is flagged
`complete: false` with `elapsed_months`. If the ticker stops trading before the horizon
(delisting, acquisition), the exit is its last close and the result is flagged
`truncated: true` — a truncated outcome is still an outcome (an acquisition at a premium
is a real return), but the reader must know.

**Returns.**

- `total_return` = adjusted exit ÷ adjusted entry − 1 — dividends reinvested, splits
  neutralised (requires the adjusted close, §4).
- `price_return` = raw exit ÷ raw entry − 1 across **split-neutralised** raw closes.
  Reported alongside because income-tilted goals care about the gap between the two.
- `annualized` = (1 + total_return) ^ (12 / elapsed_months) − 1, only when
  `elapsed_months ≥ 6` (annualising a six-week move is noise).
- `max_drawdown` = the worst peak-to-trough decline of the adjusted close between entry
  and exit (subject only; the cohort gets none — see §6).

**Benchmark.** The same computation over every member of a cohort, from the same
`as_of` for the same horizon, then: `cohort_median`, `cohort_mean`, the subject's
`percentile` within the cohort's distribution, and `excess = total_return −
cohort_median`. A thesis that made 20% in a cohort that made 30% did not work; the
excess is the number the flows' judge scores.

**Which ticker.** The company's **primary ticker active on `as_of`** (the `tickers`
table carries `active_from`/`active_to`), so a later re-ticker or a stale listing cannot
leak in — the same point-in-time discipline as facts.

---

## 3. The tool surface

```
get_forward_return(
    cik: int | None, ticker: str | None,      # one of them
    as_of: "YYYY-MM-DD",
    horizon_months: int = 12,                  # 1..60
    benchmark: str | list[int] | None = "sector",
    with_drawdown: bool = True,
)
```

`benchmark` accepts a cohort name (built-in or saved), an explicit list of ciks, the
special value **`"sector"`** — the subject's own GICS sector as a rule cohort, which is
the natural comparison and the default — or `None` for the subject alone.

```json
{
  "cik": 804328, "ticker": "QCOM", "as_of": "2024-09-30", "horizon_months": 12,
  "entry": {"date": "2024-10-01", "close": "168.84", "adj_close": "165.20"},
  "exit":  {"date": "2025-09-30", "close": "161.40", "adj_close": "161.40"},
  "elapsed_months": 12.0, "complete": true, "truncated": false, "adjusted": true,
  "total_return": -0.023, "price_return": -0.044, "annualized": -0.023,
  "max_drawdown": -0.31,
  "benchmark": {
    "kind": "sector", "name": "Information Technology", "members": 512, "measured": 497,
    "median": 0.184, "mean": 0.226, "percentile": 0.21, "excess": -0.207
  },
  "provenance": {"prices": "market", "adjustment": "provider adjusted close"},
  "notes": ["15 cohort members had no entry close within 10 days of as_of and were skipped"]
}
```

A batch form, `get_forward_returns(ciks: list[int], as_of, horizon_months, benchmark)`,
capped at 50 subjects, computes the benchmark **once** and reports each subject against
it — the shape the flows' judge needs for a field of drafts at one date.

---

## 4. Data: the adjusted close, and depth

**Adjustment.** `prices_daily` stores the raw close on purpose (market cap = close ×
shares). Returns need the *adjusted* close. Yahoo's chart payload — the one the adapter
already fetches — carries `indicators.adjclose` next to `quote.close` and, with
`events=div,splits`, the split factors; the adapter currently discards them. Proposal:

- migration **0010**: `prices_daily.adj_close NUMERIC NULL`; the unique key
  `(ticker, date)` is unchanged;
- the adapter parses `adjclose` into `DailyClose.adj_close` (a new optional field on
  the port's value object — vendor-neutral, any provider can fill it); the upsert
  writes it;
- **market cap and every existing multiple keep using `close`**; nothing that ships
  today changes meaning.

Until a row has `adj_close`, the tool falls back to raw closes and reports
`adjusted: false` in the payload and a note — never a silently split-broken number.

**Depth.** The first price sync pulled 1,100 days (≈ 3 years), so history starts around
2023-09. That bounds backtests: a 36-month horizon cannot complete for any `as_of` yet;
a 12-month horizon completes for decision dates up to ~2025-09. Two options, one
recommended:

| | rows (5,145 tickers) | first usable `as_of` for 12m / 36m |
|---|---|---|
| keep 3y | ~3.9 M (today) | 2023-09 / never yet |
| **backfill 5y** (`range=5y`, one request per ticker, one job) | ~6.5 M | 2021-09 / 2021-09 → 2024-09 |
| backfill 10y | ~13 M | 2016-09 / 2016-09 → 2023-09 |

Recommended: **5y now** — it matches the five-year facts window (a thesis evaluated at
a 2022 date has the fundamentals to be evaluated), it costs one ~2-minute fan-out like
the initial sync, and it keeps the table modest on the current disk. 10y is a later,
storage-aware decision. The backfill is the same nightly job with a `full=true` payload:
`market.sync_prices_all {full: true}` → per-ticker `range=5y` pulls that upsert both
closes. Idempotent; can run any night after 19:30 ET.

---

## 5. Honesty: what the number can and cannot claim

- **Point-in-time entry/exit** are exact: closes are knowable on their date, the primary
  ticker is the one active on `as_of`, and facts never enter this computation.
- **Survivorship in the cohort.** A cohort resolved *today* contains today's companies.
  A 2022 `as_of` benchmark therefore omits names that were in that sector in 2022 and
  have since delisted, biasing the cohort median upward. The payload says so
  (`benchmark.note`) whenever `as_of` is more than a year old. Fixing it needs a
  historical universe, which WDA does not keep; the flows' held-out rules are the
  practical mitigation.
- **Corporate actions beyond splits and dividends** (spin-offs, rights issues) are
  handled only as well as the provider's adjustment handles them. Flagged in the doc, not
  detectable per row.
- **A partial horizon is not a result.** `complete: false` outcomes are returned for
  monitoring, but the flows' judge should not score archetypes on them.

---

## 6. Design and cost

- `wda/market/outcomes.py` — pure: `forward_return`, `annualize`, `max_drawdown`,
  `cohort_stats(returns) → median, mean, percentile_of(x)`. Fully unit-tested, no I/O.
- `SqlPriceRepository` gains two **set-based** reads built on the existing
  `(ticker, date)` unique index: `first_closes_after(tickers, as_of, within_days)` and
  `last_closes_on_or_before(tickers, end)` — each one `DISTINCT ON (ticker)` query, so a
  500-name cohort is two statements, not a thousand. `closes_between(ticker, start,
  end)` serves the subject's drawdown only; the cohort never reads full series (that is
  why drawdown is subject-only).
- `SqlTickerRepository.primary_on(cik, as_of)` — the listing active on the date.
- Cohorts resolve through the existing `CohortResolver` (built-in, saved, rule, or the
  `sector` shortcut = a `gics_sector` rule on the subject's sector). The universe as a
  benchmark is allowed but slow-ish on the current disk (seconds); the payload reports
  `measured` vs `members`.
- The MCP tools are thin wrappers, like every other tool. The thesis layer does **not**
  call this in v1; attaching an outcome to an evaluation (`evaluate_thesis(as_of=…)`
  returning what happened next) is a natural follow-up once the flows use it.
- Layering: market (L3) for the maths and reads; `mcp` for the tools; the `sector`
  shortcut needs the cohort resolver, which lives in `scoring` — the tool composes them
  at the top, as `screen` does.

**Cost per call.** Subject: 3 small queries. Cohort of 500: 2 set-based queries. Cold
disk: low seconds; warm: sub-second. No writes.

**Effort.** Adjusted close (migration, port field, adapter, upsert, backfill flag, tests):
~½ day. Outcomes maths, repository reads, two tools, server registration, unit +
integration tests: ~½ day. Backfill run and live check: an evening. Guide: a card and a
line in the forward-analysis section.

---

## 7. Tests

- Unit (pure): total vs price return across a 10:1 split (adjusted vs split-neutralised
  raw); dividends widening the gap; partial horizon + `annualized` withheld under 6
  months; truncated exit; drawdown on a synthetic path; cohort median/mean/percentile
  with skipped members; entry-window rule (`no_entry` beyond 10 days).
- Integration (seeded closes): one subject + a 4-name cohort over a 12-month window;
  `sector` benchmark through a seeded GICS sector; primary ticker switch at a date; the
  `adjusted: false` fallback when `adj_close` is null.

---

## 8. Open questions (for the reader)

1. **Depth:** 5y now (recommended) — or straight to 10y and accept the larger table?
2. **Default benchmark:** the subject's GICS sector (recommended) — or none unless asked?
3. **Entry convention:** first close after `as_of` (recommended, conservative) — or the
   `as_of` close itself, which flatters every backtest by a day?
4. **Should `evaluate_thesis(as_of=…)` attach the forward outcome** once this exists, so
   a dated evaluation reads "intact then — and here is what happened"? I'd say yes, as a
   small follow-up, off by default to keep the health report a health report.
