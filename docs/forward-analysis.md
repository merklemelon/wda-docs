# Forward-Looking Analysis — Design Note

**Status:** concept / discussion (not built)
**Author:** design note, 2026-09-14
**Related:** `wda/scoring/` (the screen), `wda/enrich/` (L4 narrative), `wda/market/`
(valuation), the reserved `wda/thesis/` domain (CLAUDE.md conventions).

---

## 1. Motivation — the gap our stack has today

Everything WDA computes is **backward-looking**: point-in-time XBRL facts, trailing
margins and growth, trailing multiples (P/E, EV/EBIT, FCF yield), and a screen that
ranks *quality-at-a-reasonable-price on the numbers we already have*. That's the right
foundation, but it cannot answer the question a value investor actually faces on a
richly-valued company:

> **Tesla trades at a 300+ P/E. Is that insane, or is the market pricing products
> (Optimus, robotaxi/FSD, energy, Dojo) that have essentially zero footprint in today's
> financials?**

The trailing screen *penalizes* that multiple (low valuation percentile) and moves on.
But the whole investment question is about the **future** — revenue and margins that
don't exist yet. We need a forward-looking capability that is rigorous and honest about
what we can and can't know.

---

## 2. Two modes of forward analysis — and why we start with the *reverse* one

There are two ways to connect the future to a price:

**(a) Forward projection / pro-forma (forecast → value).** Forecast revenue, margins,
FCF for N years, discount to a present value, compare to price. This is the classic DCF.
Its fatal practical problem: it needs **forward inputs** (growth, margins, terminal
assumptions) that EDGAR does not contain (§3), so the output is only as good as
assumptions the model can't source itself. Easy to produce false precision.

**(b) Reverse-DCF / implied expectations (price → what's implied).** *Invert it.* Take
the current price as a given and **solve for the growth and margins the market is already
pricing in.** Output is a testable claim, e.g.:

> *"This price implies ~28% revenue CAGR for a decade at a 15% FCF margin. Here is the
> historical base it starts from, and here is what management says about the products
> that would have to deliver it."*

This is the intellectually honest counterpart to a forecast — the question becomes
**"what has to be true?"** rather than "what will happen?" (the framing of Rappaport &
Mauboussin's *Expectations Investing*). Crucially, **(b) needs no external forward
data** — only the current price (we have it), the historical financial base (we have
it), and DCF math. **That is why the reverse-DCF is the v1.**

Forward pro-forma (a) is still worth building later (§8), but only with the forward
drivers held as *explicit, user-owned assumptions*, never implied as facts.

---

## 3. Why this is genuinely hard to do quantitatively from EDGAR alone

EDGAR is **backward-looking by law and by design.** The specific obstacles:

- **XBRL has no forward tags.** Every fact is a reported historical/current period.
  There is no "FY2030 revenue" datum anywhere in the taxonomy.
- **No mandated financial guidance in filings.** US issuers aren't required to publish
  numeric guidance in 10-K/10-Q. When they do give quantitative guidance, it lives in
  **earnings press releases (8-K exhibit 99.1)** and **earnings-call transcripts** —
  transcripts aren't in EDGAR at all.
- **Forward-looking statements are qualitative and hedged.** MD&A (Item 7) discusses
  trends and intentions in prose, wrapped in safe-harbor language, non-standardized.
  Useful as narrative (we extract it — §4), useless as a model input on its own.
- **New products have zero financial footprint.** Optimus/robotaxi/Dojo have no line
  item, no segment, no unit economics disclosed — there is literally nothing to
  extrapolate. This is exactly where the valuation gap lives, and exactly where EDGAR
  is silent.
- **Consensus estimates are a licensed, external dataset.** The standard forward input
  (sell-side revenue/EPS estimates) comes from Visible Alpha / LSEG / FactSet / Bloomberg
  — not EDGAR — and carries its own point-in-time complexity (estimate revisions).
- **The hardest DCF inputs are judgment, not data.** Discount rate (WACC), terminal
  growth, competitive fade, and reinvestment efficiency dominate the output and cannot
  be sourced from filings.

**Net:** EDGAR can supply the *historical base* and management's *qualitative forward
narrative* — but not forward *quantities*. Any tool must be candid that the forward
numbers are **assumptions or market-implied**, never reported facts.

---

## 4. What EDGAR + WDA *can* honestly contribute

Substantial, actually — which is why reverse-DCF is viable:

| Ingredient | Source in WDA | Layer |
|---|---|---|
| Revenue, margins, FCF, FCF conversion, shares, net debt (the DCF base) | canonical financials | L3 |
| Segment revenue/margins (bottoms-up base) | 10-K segment footnotes (XBRL) | L1/L3 |
| Current price, market cap, EV | market/valuation | L3 |
| Point-in-time discipline (`as_of` on facts **and** prices) | existing | L1/L3 |
| Management's forward narrative — guidance, capex plans, strategy | `forward_statements` | L4 |
| Products in ramp, lifecycle stage | `products` | L4 |
| Dated / binary catalysts | `events` | L4 |

So we can compute a defensible historical base, invert the price into implied
expectations, and **place that number next to the qualitative drivers we already
extract** — the reverse-DCF plus its narrative context, entirely from what we have.

---

## 5. Architecture

### 5.1 Where it lives
A new analysis domain — the reserved **`wda/thesis/`** package. Like `scoring`, it sits
high in the stack (below `mcp`), **consumes** L3 financials/valuation and L4 enrichment,
is **computed on demand** (no persistence beyond optional saved scenarios), **rebuildable**,
and **point-in-time** (`as_of`). It never writes to the lower layers.

### 5.2 Components

- **Reverse-DCF engine (pure, deterministic, unit-tested).** Given a historical base
  (starting FCF or revenue+margin, shares, net debt), a current price, and a small
  **assumption set** (WACC, forecast horizon, terminal growth), solve for the single free
  variable that reconciles model value to price — e.g. the **implied revenue CAGR** (or
  implied steady-state FCF margin). Pure math over inputs; no I/O.
- **Assumption set / scenarios.** Explicit and defaulted, always overridable:
  - WACC — default from a simple CAPM (risk-free + β·ERP) or a flat house rate; overridable.
  - Horizon — default 10y explicit forecast + terminal.
  - Terminal growth — default ≈ long-run GDP (~2.5%).
  Optionally **saved** and named (reuse the cohort-store pattern — a `scenarios` table),
  so a house view is applied consistently.
- **Grounding bridge.** Assemble the L3 base + the L4 forward narrative for the name, so
  the implied number is never shown alone — it's shown with the drivers that would have
  to justify it.
- **Provenance tagging (non-negotiable).** Every number is labelled one of
  **fact | management-claim | market-implied | user-assumption**. This is the honesty
  guarantee and the single most important design rule here.
- **Forward pro-forma (v2).** A driver-based P&L where WDA supplies the base and the
  forward drivers are explicit inputs. If consensus estimates are ever wanted, they enter
  behind a `EstimatesProvider` **port** (mirroring `MarketDataProvider`) — optional,
  swappable, point-in-time-aware — never hard-wired.

### 5.3 MCP surface (read-only; no LLM budget)
- `implied_expectations(cik, as_of?, wacc?, horizon?, terminal_growth?)` → the implied
  CAGR/margin, the historical base it starts from, a sensitivity band, and a plausibility
  read; attaches the L4 forward narrative.
- `reverse_dcf(...)` / `proforma(...)` (v2) → full scenario output with provenance.
- `get_rule_fields`-style `get_assumption_defaults()` for discoverability.
All point-in-time via `as_of`; all rebuildable; all provenance-tagged.

---

## 6. Use cases — fit in the `screen → enrich → analyze` flow

WDA's funnel is: **screen** (broad quantitative rank over the universe) → **enrich**
(LLM narrative on the survivors) → **analyze** (deep, per-name work on the few that
earn it). **Forward analysis is the `analyze` tier** — the deepest, narrowest stage,
applied to names a screen + enrichment have already flagged.

Concrete flows:

1. **Rescue a "too expensive" screen result.** The screen ranks a name poorly on
   trailing multiples. Instead of discarding it, run `implied_expectations` → *"the
   market is pricing 28% CAGR for a decade"* → pull `forward_statements`/`products` →
   judge whether the pipeline plausibly supports it. This is the **qualitative override
   to the quantitative screen, done rigorously** — the screen's structural blind spot
   (it can't tell a justified high multiple from an unjustified one) filled in.
2. **The Tesla case, precisely.** Reconcile the 300 P/E to an implied growth/margin path,
   then confront it with the Optimus/robotaxi/energy narrative we already extract — turning
   a mystery multiple into a testable set of required outcomes.
3. **Point-in-time backtest of expectations.** *"What did the market imply for NVDA in
   2019, and did it deliver?"* Using `as_of` prices **and** facts (no lookahead), build
   intuition about when implied expectations were beatable — a genuinely differentiated,
   honest study only our point-in-time discipline makes possible.
4. **Cohort-level expectations scan.** Run implied expectations across a cohort (§cohorts)
   to rank names by *how much* the market is asking the future to deliver — an
   "expectations gap" screen sitting on top of the value screen.

---

## 7. Risks & mitigations

- **False precision / DCF sensitivity.** DCF value swings violently with WACC and
  terminal growth. *Mitigation:* reverse-DCF solves for one variable (less fragile than a
  forward forecast), but still show **ranges and sensitivity tables**, never a single
  point estimate; expose the assumptions on every output.
- **Spurious authority.** A clean number invites over-trust. *Mitigation:* provenance
  labels on every figure; frame outputs as *"what's implied,"* not *"the answer";* lead
  with the assumptions.
- **Thin/negative base for the names that matter most.** Early-stage, loss-making, or
  no-FCF companies (often the very growth names where forward value dominates) break a
  FCF-based DCF. *Mitigation:* fall back to revenue-driver / multiple-based framing; flag
  **low confidence** explicitly; never force a DCF where the base won't support one.
- **Base normalization is judgment-heavy.** One-offs, SBC, non-GAAP vs GAAP, restatements
  — a wrong base yields a wrong implied number. *Mitigation:* transparent, inspectable
  base; expose the exact line items used; allow overrides.
- **Point-in-time integrity.** A backtest that mixes today's facts with a past price
  launders in hindsight. *Mitigation:* enforce `as_of` on **both** facts and prices
  (discipline we already have).
- **Scope creep into forecasting.** The pull to bolt on "predictions." *Mitigation:* v1
  stays strictly *reverse* (implied expectations, no forecast); v2's forward inputs are
  explicitly user/assumption-owned and provenance-tagged.
- **External-data dependency (if consensus is pursued).** Estimates feeds are licensed,
  costly, and add point-in-time complexity (revisions). *Mitigation:* keep behind an
  optional `EstimatesProvider` port; ship nothing that *requires* it.
- **LLM-set assumptions hallucinate.** If a model fills in the drivers, it can invent
  plausible-sounding growth. *Mitigation:* assumptions are explicit inputs the user sees
  and edits; any LLM-proposed assumption is labelled a suggestion, and the engine only
  *computes* — it never *assumes*.

---

## 8. Phasing

- **v1 — reverse-DCF / implied expectations** (~1–2 days): pure engine + assumption
  defaults + `implied_expectations` tool + narrative grounding + provenance. No external
  data, no migration. Point-in-time. Its own PR with tests + deploy.
- **v2 — assumption-driven forward pro-forma** (~2–3 days): driver-based scenario P&L,
  saved scenarios (`scenarios` store, mirrors cohorts), sensitivity output.
- **v3 (optional) — consensus estimates** behind an `EstimatesProvider` port: only if an
  external feed is licensed; strictly additive.

---

## 9. Open questions

- WACC: house flat rate, simple CAPM, or per-name from betas we'd have to source?
- Terminal treatment: perpetuity-growth vs exit-multiple vs fade-to-ROIC?
- Do we want an "expectations gap" **factor** feeding back into the screen, or keep
  forward analysis strictly in the `analyze` tier (my lean: keep it in `analyze` — it's
  judgment-heavy and doesn't belong in a deterministic cross-sectional rank)?
- Normalization policy for SBC and non-GAAP in the base.

---

## 10. Calibration findings (2026-09-14, after the mid-cycle base shipped)

WACC 10% / 10-yr / terminal 2.5%; band = WACC 8–12% × terminal 2–3%.

| | Price | FCF latest | FCF mid-cycle | Implied (latest) | Implied (mid-cycle) | Note |
|---|---:|---:|---:|---:|---:|---|
| TSLA | 358.97 | 6.2B | 4.3B | +40.2% | +45.9% | headline holds: a decade of ~40%+ |
| NVDA | 210.96 | 96.7B | 96.7B | +21.1% | +21.1% | bases identical |
| MSFT | 505.41 | 67.0B | 84.4B | +21.9% | +18.7% | |
| QCOM | 180.15 | 12.8B | 12.7B | +4.9% | +5.0% | the value case |
| META | 665.60 | 46.1B | 65.3B | +16.3% | +11.6% | |
| AAPL | 333.08 | 98.8B | 108.1B | +20.5% | +19.2% | |
| JPM | 356.23 | n/a | n/a | — | — | bank: `missing=fcf` (graceful); ticker fix confirmed (`JPM`, not AMJB) |
| SPCE / INTC | | negative | n/a | — | — | `no_positive_base` — honest, unanswerable |
| **MU** | 924.03 | 1.67B | **0.18B** | +57.8% | **+98.4%** | **CYCLICAL** — see below |

**What it validated.** On stable names the two bases agree and nothing is flagged; the
mechanism is sound. The FY label now derives from `period_end`. Banks and FCF-negative
names fail gracefully.

**The honest surprise — median-of-≤5-FY is not a universal "mid-cycle".** For Micron the
normalization made the number *worse* (58% → 98%, pinned near the +100%/yr ceiling),
because the five-year window is dominated by the FY2023–24 memory downturn: the *median*
margin sits **below** the recovering latest year, so "normalizing" shrank the base. The
`cyclical` flag fired correctly (the bases disagree materially) — that signal is the
valuable part — but the *direction* of a median-based normalization is wrong whenever
the window is phase-skewed. Two conclusions:

1. No single normalization is neutral; it is a judgment. Keep showing **both** bases +
   the flag (shipped). Consider a third, explicitly optimistic **peak-margin view** (best
   margin in the window × current revenue) so the three implied growths *bracket* the
   answer for cyclicals, and/or a full-cycle window (7–10 FY) where history allows.
2. Micron's honest reading is not a bug: at $924 the price requires a **supercycle** —
   roughly 50–100%/yr FCF growth for a decade off a still-depressed base. The tool's job
   is to say that plainly (band near the ceiling + `CYCLICAL`), not to hide it.

---

## 11. v1.1 — revenue mode and the buyback question (2026-09-14)

**Revenue mode (FCF-negative names).** `implied_expectations(target_fcf_margin=…)` assumes
the company earns the target FCF margin on revenue *from year one* and solves for the
**revenue** growth the price implies. When no target is given it uses the company's own
best positive historical margin (provenance `fact_peak_margin`), and when none exists it
asks for one rather than inventing a default. Because FCF_t = revenue_t × m, this is the
FCF solver on a base of revenue × margin; there is **no margin ramp**, so for a loss-making
turnaround the read is *optimistic* and is labelled so in the reading.

**Buybacks — a correction to §7's framing.** A total-FCF DCF is **not** inflated by
buybacks: equity value is the PV of *all* future FCF less net debt, divided by *today's*
shares; future buybacks only redistribute that FCF among remaining holders (they
concentrate value, they don't create it). So the implied total-FCF growth needs no
"correction". What investors actually want is the **per-share translation** — with a net
buyback yield *b*, total growth *g* reads as (1+g)/(1−b) − 1 per share. v1.1 surfaces the
historical net share-count shrink (`buyback_yield`, ≤5-FY CAGR of diluted shares) and that
translation in the `per_share` block and the reading. Apple at ~20%/yr total FCF growth
with ~3%/yr buybacks is ~24%/yr per share — the same claim, stated the way EPS-growth
thinking expects.

**Split guard (found by the live check).** Virgin Galactic's 1-for-20 reverse split showed
up as a +61%/yr "buyback yield", and NVIDIA's 10:1 split would read as massive issuance. A
share count moving more than ~30%/yr is a split, reverse split or transformative deal, not
a buyback program — the yield (and the per-share translation) is withheld in that case
rather than misattributed. Point-in-time share facts are not reliably split-adjusted.
