# Technical Analysis in the Stack — Design Note

**Status:** **v1 and v2 shipped 2026-09-16** (§6 phasing) — `momentum_12_1` as a factor
(#84, #85) and `adv_63` liquidity plus the `prices_daily.volume` column (#86, #87). Live
coverage on 2026-09-24: momentum on 3,612 of 3,712 snapshots, ADV on 3,704. Both ship at
**weight 0**, so the flagship screen is unchanged until a caller passes `weights`; both are
`get_rule_fields` predicates (the liquid cohort was `adv_63 gte 5_000_000`).
**v3 shipped 2026-09-24**: a `price_metric` thesis check over `close`, `realized_vol`,
`dist_from_high` and `max_drawdown`, computed on demand from the close series (no
migration — these are per-company reads, not snapshot factors) and point-in-time via a new
`until` bound on the price read. Values are **fractions**, deliberately, against the
screen's percent-valued `momentum_12_1`. **v4 (divergence) is not built.** §3
(chart-reading TA) remains out of scope by decision.

A related surface gap closed on 2026-09-23: `get_prices` now returns `adj_close` and
`volume` per bar with up to ~5 years of history, so a caller can do the analysis this note
declines to do server-side.
**Author:** design note, 2026-09-14
**Related:** `wda/market/` (`prices_daily`), `wda/scoring/factors.py` (the factor
registry), `wda/scoring/rules.py` + cohorts (rule vocabulary), [thesis.md](thesis.md)
(`Check` types), [forward-analysis.md](forward-analysis.md) (price → implied expectations).

---

## 1. The question, and the stance

Does technical analysis belong in a fundamentals-first, value-oriented stack? The honest
answer: **a narrow, quantitative slice of it — yes; chart-reading TA — no.** The whole
value hinges on drawing that line clearly.

- **Discretionary TA** — chart patterns (head-and-shoulders, Fibonacci, Elliott waves),
  oscillators as buy/sell triggers, short-horizon signals. **Out of scope.** Poorly
  validated, philosophically opposed to WDA's long-horizon value frame, and it would
  dilute the product's identity.
- **Quantitative price signals** — momentum, volatility, liquidity, price-vs-fundamentals
  divergence. **In scope**, subordinate to the fundamental thesis: signal and risk/execution
  *context*, never a standalone predictive layer.

Guiding principle: **price data serves the fundamental case; it never overrides it.** Also
worth noting price is *already* central to the stack — it's the input to the reverse-DCF's
implied expectations. We're not adding price analytics; we're extending them.

---

## 2. What fits — legitimate uses, ranked by strength

### 2.1 Momentum as a factor (strongest case)
Cross-sectional momentum — trailing 12-month return skipping the most recent month — is
one of the most robust, replicated return premia in the literature (Jegadeesh–Titman; AQR
*"Value and Momentum Everywhere"*). Critically it is **complementary to value**: cheap
names are often falling, so value and momentum diversify each other — running both is
well-supported, not contradictory.

```
momentum_12_1 = close[t - 21d] / close[t - 273d] - 1     # ~12m return, skip last ~1m
```

Direction: higher is better. Home: a new **`momentum` bucket** in `factors.py`, beside
`quality` and `valuation`.

### 2.2 Liquidity (average dollar volume)
A practical, price-derived filter — *"can I actually trade this?"* Directly relevant to the
"investable universe" question (see the [broad-l3 backfill] discussion: the screenable set
is not the same as the *tradeable* set).

```
adv_63 = mean( close * volume ) over the trailing ~63 trading days
```

Home: a **cohort definition** (an investable/liquid cohort), expressible as a rule once
`adv` is a rule field. **Requires a schema change** — `prices_daily` currently stores only
`close`; ADV needs a `volume` column (the Yahoo provider already returns volume). Small
migration.

### 2.3 Volatility / drawdown / distance-from-high (risk & thesis context)
Not price prediction — *risk framing* and **thesis tripwire inputs**.

```
realized_vol   = annualized stdev of daily log returns over trailing 90/180d
dist_from_high = close / max(close over trailing 252d) - 1      # % below 52-week high
drawdown       = close / running_peak - 1
```

Home: `Check` types in the thesis object model (e.g. a condition *"price hasn't broken
below $X"* or *"realized vol stays under Y"*), and position-sizing context.

### 2.4 Price-vs-fundamentals divergence (the most WDA-native use)
Where price is most useful *to a fundamentals investor*: flag when fundamentals improve
while price falls (potential value setup) or price rips while fundamentals stagnate
(potential froth). It anchors price action **to** fundamentals rather than treating it as
self-predictive.

```
divergence = sign(trailing fundamental trend: revenue/margin) vs sign(trailing price trend)
```

Home: an `analyze`-tier read (a small service that compares a fundamental trend from L3 to
a price trend from L3 market).

### 2.5 Price is already analytical (reverse-DCF)
`implied_expectations` uses current price as its core input. Price already has a first-class,
non-TA role — as valuation, not charting.

---

## 3. What doesn't fit, and why

Discretionary chart patterns, candlestick signals, indicator-cross triggers (MACD/RSI
buy-sells), and any sub-quarter timing signal. Reasons: weak/failing out-of-sample
validation, high researcher-degrees-of-freedom (easy to overfit), a horizon mismatch with
a value thesis, and identity dilution. If ever wanted, they'd live strictly as *optional,
labelled, backtested* signals — never in the default screen.

---

## 4. Architecture anchoring

Everything rides on `prices_daily` (L3 market) we already sync; these are cheap derived
metrics, computed point-in-time (from the price series `as_of` a date — no lookahead,
consistent with the rest of the stack).

| Signal | Extension point | Notes |
|---|---|---|
| momentum (12-1), realized vol | new **`momentum` bucket** in `wda/scoring/factors.py` | re-weightable like every factor; filterable via the `get_rule_fields` vocabulary; point-in-time |
| liquidity (ADV) | a **cohort** definition / rule field | needs a `volume` column on `prices_daily` (small migration) |
| vol / drawdown / distance-from-high | **thesis `Check` types** | monitorables/tripwires in [thesis.md](thesis.md) |
| price-vs-fundamentals divergence | an **`analyze`-tier** service | compares L3 fundamental trend to L3 price trend |
| price → implied expectations | already the forward engine | not TA, but the existing analytical use of price |

Because momentum/vol slot into the factor registry, they inherit everything the screen
already does — percentile ranking within a cohort, custom weights, point-in-time, and rule
predicates — for free. No new subsystem; a new bucket and a couple of derived series.

---

## 5. Risks & cautions

- **Momentum crashes.** Momentum has real tail risk — sharp reversals at market turns.
  It's a *portfolio/ranking-level* premium, not a per-name timing oracle.
- **Value vs momentum tension on single names.** They pull against each other on any one
  cheap-and-falling stock; the benefit is in combining them across the ranking, not
  reconciling them on one name. Keep momentum a *modest, tunable* weight — the default
  screen should stay value-and-quality led, with momentum opt-in (or low default) so we
  don't quietly change the screen's character.
- **Overfitting / signal creep.** The curated, factor-shaped approach (a handful of
  well-motivated signals) is the guardrail against an ever-growing indicator zoo. Any new
  signal must be motivated and, ideally, point-in-time backtested before it earns a
  default weight.
- **Data completeness.** ADV needs volume (not yet stored); realized vol/momentum need
  enough price history per name (the 1,100-day sync depth covers it).

---

## 6. Recommendation & phasing

- **v1 — momentum factor** (~½ day): `momentum_12_1` (and optionally `realized_vol`) as a
  new `momentum` bucket in `factors.py`, low/zero default weight so the default screen is
  unchanged and users dial it in via `weights` (or a "value + momentum" preset). Pure math
  on the existing price series; its own PR + tests + deploy.
- **v2 — liquidity cohort** (~½ day + migration): add `volume` to `prices_daily`, compute
  `adv`, expose it as a rule field → an investable/liquid cohort. Cleans up the "tradeable
  vs screenable universe" question.
- **v3 — thesis technical checks** ✅ *shipped 2026-09-24*: vol / drawdown /
  distance-from-high / absolute close as a `price_metric` `Check` type, plus the §2.3
  example condition "price hasn't broken below $X". Pure math in
  `wda/market/technicals.py`; `max_drawdown` is the only order-sensitive one and is
  silently wrong on a reversed series, so the chronological convention is pinned by a test.
- **v4 (optional) — divergence read**: an `analyze`-tier price-vs-fundamentals signal.

Highest value / lowest cost first step is **v1 momentum** — best-validated, complements the
value tilt, and we already have the prices.

---

## 7. Open questions

- Default momentum weight: zero (pure opt-in) vs a small non-zero (ships a value+momentum
  blend)? Lean: zero default + a named preset, so the flagship screen stays value-led.
- Momentum definition: 12-1 only, or also shorter/longer variants and a risk-adjusted
  (vol-scaled) momentum?
- Should liquidity be a hard universe filter (exclude illiquid names from the rankable set)
  or just a factor/cohort the user opts into? Lean: a cohort, not a silent exclusion.
