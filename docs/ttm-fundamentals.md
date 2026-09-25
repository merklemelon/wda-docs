# Data Readiness — TTM Fundamentals and Self-Checking Reads

**Status:** proposal, 2026-09-18 (not built) — approved to implement
**Related:** [fundamentals-and-valuation.md](fundamentals-and-valuation.md) (the metric
catalog and the standalone-quarter derivation), [architecture.md](architecture.md) §L5
(factor snapshots), [thesis.md](thesis.md) §checks (the quarterly window that already
exists), [progress.md](https://github.com/merklemelon/wda) (the working log lives in the private app repo) (the 10-Q narrative work of 2026-09-17).

One goal in two halves: **an MCP user should get the data they expect without having to
verify it.** Part A fixes a ranking that is computed on the wrong basis. Part B makes
every answer state what it was built from, and keeps the corpus current without being
asked. They are independent builds and can ship in either order.

- **Part A (§1–§7)** — TTM fundamentals, so the screen can see the last three quarters.
- **Part B (§8–§13)** — self-healing coverage and a `coverage` block on every read.

---

# Part A — TTM Fundamentals

## 1. The defect

The screen ranks on fundamentals drawn from each company's **latest complete fiscal
year**. `fiscal_rows()` asks the metrics layer for an annual series and takes the most
recent row; `fundamentals_from_rows()` derives the quality block from it; the multiples
divide the nightly close by figures from that same year. Nothing in the ranking path
reads a 10-Q.

For a December filer screened in September, the denominator is nine months old and three
reported quarters are invisible. That is not merely stale, it is **wrong in a predictable
direction**:

> A company reports a bad quarter. The price falls. The annual denominator does not move.
> The multiple improves. The name ranks **cheaper** on the bad news.

The screen promotes deteriorating businesses. A value screen with this property is not
conservative; it is systematically adversely selected.

**The live example.** AECOM (CIK 868857), 2026-09-18. Its FY26 Q3 (filed 2026-08-11)
carried a $337M pre-tax charge on a delayed Construction Management project and a cut to
FY2026 guidance (adjusted EPS $3.95–4.15). Both facts are in WDA today: the charge and
the guidance are in `events` from the 8-K of 2026-08-10, the MD&A is indexed, the
quarterly facts are in `xbrl_facts`. Its **screen factors still describe fiscal 2025**
and will until the FY26 10-K lands in November.

**Scope of the blind spot.** Every ranking path shares it:

| Path | Fundamentals basis | Sees a 10-Q? |
|---|---|---|
| `screen`, `get_score` | latest complete fiscal year | no |
| `get_valuation` | "latest annual figures and the latest close" | no |
| cohort rules (`ev_ebit`, `fcf_yield`, …) | the same snapshot | no |
| `implied_expectations` | five complete fiscal years | no |
| `get_financials(freq="quarterly")` | derived standalone quarters | **yes** |
| `get_facts` | raw | **yes** |
| thesis `financial_metric` | `window: "quarterly"` (opt-in; default annual) | **yes, if set** |
| narrative (`events`, `forward_statements`, risk updates) | 10-K + 10-Q + 8-K | **yes** |

So the divergence *is* detectable today — but only on a company you have already singled
out. The one component whose job is to tell you **where to look** is the blind one. That
inversion is the argument for doing this now.

---

## 2. Why TTM is the baseline, not an enhancement

Last-twelve-months is the default basis for multiples in every institutional system
(Bloomberg, CapIQ, FactSet). An analyst handed a ranking built on a fiscal year that
closed nine months ago would not read it as conservative. The professional ladder is:

1. **NTM** (next twelve months, on consensus estimates) — the buy-side default.
   **Unavailable to us:** WDA has no estimates feed and adding one is a vendor decision,
   not a derivation. Out of scope.
2. **TTM / LTM** — the achievable standard, derived entirely from data we already hold.
3. **Latest fiscal year** — where we are.

This proposal moves us from 3 to 2. It does not attempt 1.

---

## 3. What we can derive, and what we cannot

The metrics layer already tags every metric `additive` and derives standalone quarters by
differencing year-to-date cumulatives (fiscal Q4 = FY − 9M). That machinery is the whole
basis for TTM; no new parsing is required.

**Flows — sum the four most recent standalone quarters.** Revenue, cost of revenue,
gross profit, operating income, net income, R&D, SG&A, income tax, operating cash flow,
capex (hence FCF). These are `additive=True`; the quarter series already exists.

**Stocks — take the latest quarterly instant, no summing.** Assets, liabilities, equity,
cash, debt, inventory, current assets/liabilities. Balance-sheet items are point-in-time
by nature, so `current_ratio` and `debt_to_equity` become current the moment a 10-Q
lands, with no TTM arithmetic at all. *This is the cheapest win in the proposal.*

**Per-share — must be reconstructed, never summed.** `eps_basic`, `eps_diluted` and
`shares_diluted` are `additive=False` and, by deliberate design, are **never
differenced** — only directly tagged 3-month facts are kept. A TTM EPS is therefore

```
eps_ttm = net_income_ttm / shares_diluted_latest
```

That is the conventional treatment, but it is a **modelling choice we are making**, not a
figure the filer reported. It must be labelled as derived wherever it surfaces, the same
way derived gross profit already is.

**Ratios** (margins, intensities, FCF margin) are computed from the TTM flows above, not
averaged across quarters.

---

## 4. Design

### 4.1 Snapshot shape

`factor_snapshots` gains a parallel fundamentals block rather than replacing the annual
one. Keeping both is what makes §4.3 nearly free.

| Column | Meaning |
|---|---|
| `quality` | unchanged: the latest complete fiscal year (kept for continuity and audit) |
| `quality_ttm` | the same factor keys, computed on the TTM basis |
| `ttm_period_end` | the end of the most recent quarter in the window |
| `ttm_quarters` | how many standalone quarters were actually summed (4 = clean) |
| `ttm_basis` | `"ttm"` \| `"annual_fallback"` — what the factors were actually built from |

`fiscal_period_end` and `fundamentals_at` keep their current meaning.

### 4.2 Which basis the scorer uses

The scorer reads **TTM when `ttm_quarters == 4`, else falls back to the fiscal year**,
and every result says which. A company with a gap in its quarterly record (a recent IPO,
an irregular filer, a fiscal-year change) is ranked on the annual basis and labelled
`annual_fallback` rather than being dropped or silently mixed. Mixing bases *within* one
company's factor set is never allowed.

### 4.3 The divergence flag

With both bases stored, the analyst signal is a subtraction:

```
ttm_vs_fy[metric] = (ttm - fy) / |fy|
```

Surfaced per company on `get_score` and as an optional rule field, this is a
**fundamental-trend** read: how much the recent three quarters differ from the year-ago
three. It is the flag this proposal was prompted by.

**Its honest limitation:** a TTM window dilutes one dramatic quarter across three normal
ones. AECOM's $337M charge against ~$16B of annual revenue barely moves a TTM margin. The
divergence is a good *ranking* input and a **weak alarm**. The sharp alarm is a single
quarter against its year-ago quarter, which the thesis layer already supports via
`window: "quarterly"` with `hysteresis_periods`. The two are complements; the proposal
recommends documenting that pairing rather than building a third mechanism.

### 4.4 Refresh cadence

No new scheduling. The facts sync already chains `scoring.snapshot` for the company whose
facts changed, and since 2026-09-17 a newly discovered 10-Q is fetched, enriched and
fact-refreshed on arrival. Rebuilding on that same hook means **a 10-Q moves the multiples
the day it lands**. The nightly `revalue_all` continues to re-derive multiples from stored
fundamentals and the latest close, unchanged.

### 4.5 Point-in-time

`as_of` screens recompute from facts on demand. They must use the **same TTM recipe** with
the same `filed_at` cut, or a backtest silently disagrees with the live screen. This is the
part most likely to be got wrong and needs a test asserting the two paths agree for a
company on a date.

---

## 5. What this does not fix

- **One-off contamination.** An impairment or settlement sits in TTM for four quarters,
  making a name look expensive on TTM earnings, then rolls off abruptly and the multiple
  "improves" with no business change. This is the standard critique of LTM screening and
  it cuts both ways. We should state it in the Field Guide rather than bury it. A
  normalized or median-of-five-years basis is the eventual answer and is out of scope here.
- **Guidance and forward-looking change.** A cut to next year's guidance moves no reported
  number. That lives in the narrative layer (`forward_statements`, `events`), which since
  2026-09-17 reads 10-Qs and 8-Ks. TTM does not substitute for it.
- **Price-side staleness.** Separate defect, separately tracked: ~17% of tickers miss the
  same-day bar and heal the next night (progress.md, 2026-09-18).
- **momentum_12_1 / adv_63.** Price-derived, unaffected by any of this.

---

## 6. Cost and shape of the work

Part A is three PRs (numbered 3–5 in the combined plan, §13):

1. **Derivation.** `ttm_from_quarters()` in `wda/scoring/snapshots.py` plus the metric
   catalog work for the per-share reconstruction. Pure functions, unit-tested against
   hand-checked fixtures (a clean 4-quarter filer, a fiscal-Q4-derived case, a gap case,
   an IPO with two quarters). No I/O, no migration.
2. **Persistence and scoring.** Migration for the new columns, snapshot builder writes
   both bases, scorer selects with fallback, `basis` and `get_score` report which was
   used. Full rebuild of 3,712 snapshots at the end (the streaming `build_all` path,
   which last ran comfortably post-upgrade).
3. **Surface.** `ttm_vs_fy` on `get_score`, optional rule field, server instructions,
   Field Guide section including the one-off caveat.

The point-in-time parity test (§4.5) belongs in the persistence PR and is the one piece of
real risk in either part. See §13 for the combined ordering with Part B.

---

## 7. Decisions still open

The build is approved (2026-09-18). The current behaviour is not a gap in coverage, it is
a ranking that moves the wrong way on new information, and every input needed to fix it
is already in the database. These five choices remain, each with a recommendation; none
blocks starting Part B.

1. **Default basis.** TTM with annual fallback (recommended), or keep annual as the
   default and expose TTM as an opt-in `basis` parameter for one release?
2. **Backfill depth.** Rebuild all 3,712 snapshots at once, or TTM-ify on next touch and
   let the universe converge over a quarter?
3. **`ttm_vs_fy` exposure.** A scored factor with a real weight, a zero-weight rule field
   (like `adv_63`), or reporting-only on `get_score`? Recommendation: zero-weight rule
   field first, earn a weight later with evidence.
4. **`eps_ttm`.** Ship the reconstruction, or omit per-share TTM entirely in v1 and rank
   P/E on the annual basis with a label? Recommendation: ship it, labelled derived.
5. **Retention of the annual block.** Keep `quality` forever (recommended, it is the audit
   trail and the divergence input), or drop it once TTM is trusted?

---

# Part B — The App Does the Checking

## 8. Three failure modes, conflated

"Am I getting the data I expect?" hides three distinct failures. They need different
fixes and it is worth not confusing them.

| # | Failure | Example | The fix is |
|---|---|---|---|
| 1 | **Data missing, nobody knew** | 28 companies had a 10-K and nothing else until 2026-09-18 | self-healing (§9) |
| 2 | **Present, but a different basis than assumed** | screen ranks on a fiscal year 9 months old; ~17% of prices a day behind | fix the basis (Part A), then disclose (§10) |
| 3 | **Question and corpus mismatch** | a thematic search reads as a market scan; it covers 34 of 5,318 companies | scope disclosure (§10) |

The third is the most dangerous and the one we disclose *nothing* about today. Ask
"which companies flag tariff exposure" and `search_enrichment` returns confident,
well-formed hits drawn from 0.6% of the universe, with nothing in the response saying so.
Failures 1 and 2 produce answers that are late or off-basis. Failure 3 produces an answer
that is **plausible and unrepresentative**, which is worse, because nothing about it
invites a second look.

## 9. Self-healing: the coverage reconciler

Arrival-enrichment (2026-09-17) only catches filings that arrive *after* it went live. It
has no notion of catching up, which is why a one-off manual backfill was needed on
2026-09-18. Without a reconciler that gap reopens the first time a job fails, a company is
added, or a deploy lands mid-chain.

**`enrich.reconcile_coverage`**, scheduled nightly after the thesis re-check:

1. For every company with narrative enrichment on record, resolve its fiscal-year window
   (latest 10-K, subsequent 10-Qs, recent 8-Ks) — the same `_fiscal_year_filings` the
   planner already uses.
2. Keep those with any unprocessed applicable target.
3. Enqueue each company's plan, **bounded**: at most `RECONCILE_MAX_COMPANIES` per night
   and stalest-first, so a large backlog drains over several nights instead of
   monopolising the worker or the budget.
4. Idempotent by construction: a company that is current plans nothing.

Design notes:

- **Bounded, not greedy.** The nightly cap is what makes this safe to leave running
  unattended. Stalest-first ordering (oldest newest-processed filing) means the cap
  always spends on the most valuable work.
- **Budget-aware.** It should stop enqueueing when the projected spend would exceed the
  remaining daily ceiling rather than letting the breaker fail jobs, which leaves a
  company half-enriched and a failure in the trail. *Failing cleanly beats failing
  loudly.*
- **Observable.** Log companies reconciled and filings enqueued; surface a
  `coverage_debt` count on `get_ingest_status` so the backlog is visible without a query.

This is the highest-value item in the proposal. It converts a property we currently
achieve by hand into one the system maintains.

## 10. A `coverage` block on every analytical read

Today only `get_enrichment` and `get_risk_themes` carry freshness (`as_of_filing`,
`newer_filings_unprocessed`, `fix_with`). Every other analytical surface is silent.

| Tool | The block should say |
|---|---|
| `search_enrichment`, `search_filings` | corpus size (companies, filings), that it is a **curated subset** not the universe, how many hits came from filings older than the company's latest, `fix_with` to widen |
| `screen`, `get_score` | basis as a **range** (`price_date` min/max) with a count behind the max, the coverage distribution, and which fundamentals basis was used (TTM vs annual fallback, per Part A) |
| `get_financials`, `get_facts` | whether a newer filing exists on record that these figures do not reflect |
| `implied_expectations`, `get_valuation` | the fiscal year the figures come from and its age |

Three rules keep it useful rather than noisy:

- **Embed, never add a tool.** The nudging failure of 2026-09-16 taught us that anything
  the client model must *remember* to call is a thing it will skip. A `get_coverage` tool
  would be skipped exactly when it mattered. Coverage has to arrive unbidden, inside the
  answer the model already wanted.
- **Complete says "complete", in one word.** Only exceptions get prose. A response that
  always carries a paragraph of caveats trains both the model and the reader to skim
  them, at which point disclosure is indistinguishable from silence.
- **Every disclosure carries its fix.** A gap the reader cannot close is just an apology.

## 11. Server instructions

The read-side counterpart to the SET AND FORGET section. Never present a search or a
screen without stating its corpus and basis: *"across the 34 enriched companies"*, not
*"companies flagging X"*. When a `coverage` block reports a gap, say so in the answer and
offer its `fix_with` — do not silently rank or summarise around it.

## 12. What this cannot do

- **Over-disclosure defeats itself.** Covered above; the mitigation is the one-word rule.
- **It cannot check semantics.** The app can verify presence, freshness and scope. It
  cannot verify that the user's question means what the data means. It can report that
  figures are annual; it cannot know the reader assumed trailing twelve months. That gap
  closes by **fixing the basis (Part A)**, not by describing it more carefully.
  Disclosure is the patch for gaps you cannot close; closing them is strictly better.
- **It cannot make a curated corpus representative.** Saying "34 companies" is honest, but
  a thematic search over 34 names is still a different instrument from a market scan. The
  eventual answer is wider enrichment coverage, which is a cost decision, not a design one.

## 13. Sequencing

Five PRs, each independently green. Part B first: it is smaller, has no migration, and
stops the drift that would otherwise re-accumulate underneath Part A.

| PR | Scope | Migration |
|---|---|---|
| 1 | `enrich.reconcile_coverage` job + schedule + `coverage_debt` on `get_ingest_status` | no |
| 2 | `coverage` blocks on search / screen / financials; screen `basis` as a range; price catch-up pass before `revalue_all` | no |
| 3 | TTM derivation (`ttm_from_quarters`, per-share reconstruction) — pure functions | no |
| 4 | TTM persistence + scoring with annual fallback; point-in-time parity test; full rebuild | **yes** |
| 5 | `ttm_vs_fy` surface, server instructions, Field Guide | no |

PR 4 carries the only real risk (point-in-time and live screens must agree) and the only
migration. PRs 1 and 2 are safe to ship the same day.
