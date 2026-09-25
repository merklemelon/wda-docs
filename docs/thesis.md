# The Thesis Layer — Design Note

**Status:** phase 1 implemented 2026-09-15 (PRs #73 engine, #74 persistence + tools); v2/v3 per §7.
**Updated 2026-09-24:** a sixth check type, `price_metric` (TA v3, #118), binds price-derived
risk to a condition; `roe`/`roa`/`roce` (#119, #120) are usable through `financial_metric`;
§4.3 works all three. §4.4 documents the `avoid`/`watch` stances and the convention for a
thesis that keeps you out.
Persistence landed as migration **0009** (0008 went to factor snapshots). Deviations from §7: the
`implied_expectations` check shipped in v1 (forward v1 existed); monitoring is a nightly
`thesis.evaluate_all` at 20:00 ET with `thesis_evaluations` history — the filing-triggered hook is v2.
**Author:** design note, 2026-09-14
**Related:** [forward-analysis.md](forward-analysis.md) (the `analyze` tier feeding a
thesis), `wda/scoring/` (screen + the rule-predicate pattern this reuses),
`wda/enrich/` (narrative checks), `wda/market/` (valuation checks), the reserved
`wda/thesis/` domain, [decisions.md](decisions.md) (the macro sources ADR, 2026-09-25).

---

## 1. Recap — a thesis is a living, checkable object

A thesis here is **not** a write-up. It is a **structured, falsifiable, machine-checkable
object**: a position plus the evidence-bound conditions that must hold for it to be right,
and the tripwires that declare it broken. Its power is that WDA can **re-check it against
new data, point-in-time, indefinitely** — the discipline most investors lack because their
thesis lives in a doc no process ever revisits.

The enabling idea: **every condition binds to a WDA capability.** "Margins hold" = a
`get_financials` check. "No new structural risk" = a `get_risk_themes(is_new)` check. "The
market isn't asking too much" = an `implied_expectations` check. A thesis is a set of live,
thresholded queries over the stack.

---

## 2. How each idea anchors to WDA architecture

The thesis layer invents very little — it **composes** what's already here. The mapping:

| Thesis concept | Anchors to | Existing pattern it reuses |
|---|---|---|
| Point-in-time evaluation & backtest | `filed_at` on facts + `as_of` on derived reads | rule 4 (no-lookahead); already enforced in facts/valuation/screen |
| Condition **binding grammar** (field/op/value → a check) | the cohort **rule predicate** | `wda/scoring/rules.py` — same operator set (`gt/gte/lt/lte/eq/ne/in/not_in`), same validate/evaluate split |
| Quantitative checks (margin, growth, balance sheet) | canonical financials | L3 `get_financials` / `ScreenService.factor_values` |
| Valuation checks (P/E, EV/EBIT, FCF yield) | market/valuation | L3 `get_valuation` |
| Relative checks (factor percentile vs a peer set) | the screen + cohorts | `get_score` within a `cohort` |
| Narrative checks (a risk emerged / a product shipped) | LLM enrichment | L4 `get_risk_themes(is_new)`, `search_enrichment` |
| Expectational checks (implied vs guidance) | the forward engine | `wda/thesis` reverse-DCF + L4 `forward_statements` |
| Price-risk checks (level, volatility, drawdown) | the price series | `wda/market/technicals.py`, computed on demand and point-in-time |
| Event checks (a catalyst by/after a date) | enrichment events | L4 `events` |
| Macro checks (rates/energy/regulation) | the macro layer | `wda/macro/` **ships**; the *check type* binding a series to a condition does not |
| Persisted thesis store | a domain with a repo + migration | mirrors the **cohorts** store (`wda/cohorts/`) |
| Define = write with a safety gate | preview → confirm | rule 9 (two-step writes) |
| Filing-triggered re-evaluation | job queue + scheduler + filings | the existing worker/scheduler; a thesis "subscribes" to its subject |
| Provenance (fact / claim / implied / assumption) | shared with forward-analysis | the honesty guarantee from [forward-analysis.md](forward-analysis.md) |
| "Unknown" when data is missing | coverage honesty | same discipline as the screen's coverage counts |

Two anchors are worth stressing:

- **A thesis condition is the single-company dual of a cohort rule.** A cohort *rule*
  filters the universe by factor predicates; a thesis *condition* checks one company by a
  tool-bound predicate. Same grammar, same operators, same validate-then-evaluate shape —
  so the engine, the tests, and the MCP ergonomics all rhyme with something that already
  ships.
- **The evaluation engine calls services, not MCP tools.** `wda/thesis` sits *below* `mcp`
  in the layer order, so a check resolves through the same services the tools wrap
  (`ScreenService`, valuation, enrich repos, the forward engine) — the MCP tools are thin
  wrappers over the thesis service, exactly like every other layer.

---

## 3. The typed object (scaffolding)

Illustrative — Pydantic/dataclass shapes and the persisted tables. The heart is
**`Condition`**, a unified checkable unit (a "claim" and a "tripwire" are the same type
with opposite `polarity`).

### 3.1 Core types

```python
class Provenance(str, Enum):
    FACT = "fact"  # reported in a filing
    MANAGEMENT_CLAIM = "mgmt_claim"  # said by the company (guidance/MD&A)
    MARKET_IMPLIED = "market_implied"  # derived from price (reverse-DCF)
    USER_ASSUMPTION = "user"  # the analyst's own input


class Stance(str, Enum):
    LONG = "long"
    SHORT = "short"
    AVOID = "avoid"
    WATCH = "watch"


class Conviction(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"


class Polarity(str, Enum):
    MUST_HOLD = "must_hold"
    MUST_NOT_OCCUR = "must_not_occur"


class Severity(str, Enum):
    CORE = "core"
    SUPPORTING = "supporting"
    INFO = "info"


class Status(str, Enum):
    SUPPORTED = "supported"
    WEAKENING = "weakening"
    BROKEN = "broken"
    UNKNOWN = "unknown"


@dataclass
class Assumption:
    label: str  # "WACC", "terminal growth", "rates stay < 5%"
    value: str
    provenance: Provenance
    note: str = ""


@dataclass
class Condition:
    statement: str  # human-readable, e.g. "gross margin stays above 55%"
    polarity: Polarity  # must_hold (a claim) | must_not_occur (a tripwire)
    severity: (
        Severity  # core => failure breaks the thesis; supporting => weakens; info => tracked only
    )
    check: Check | None  # None => a "soft" condition: surfaced, never auto-scored
    hysteresis_periods: int = 1  # require N consecutive breaches before flipping (anti-flap)
    # ---- computed at evaluation, not stored on the definition ----
    status: Status = Status.UNKNOWN
    last_value: str | None = None
    tripped_at: date | None = None


@dataclass
class Thesis:
    slug: str  # "qcom-cheap-quality-2026"
    cik: int
    stance: Stance
    horizon_months: int
    conviction: Conviction
    summary: str
    valuation_anchor: dict  # {"method":"reverse_dcf","implied_cagr":0.28,"target":...}
    assumptions: list[Assumption]
    conditions: list[Condition]
    # ---- computed ----
    status: Status = Status.UNKNOWN  # roll-up over conditions
```

### 3.2 The `Check` vocabulary — the binding grammar

A check is a **tagged union** whose types map to services. Deliberately a *curated
whitelist* (like `rule_fields()`), so the model has an easy, safe target and the engine
knows how to resolve each one. Every check carries a `comparator` from the shared operator
set + a typed `threshold`.

**Shipped** (`CHECK_TYPES` in `wda/thesis/checks.py` — these six, and only these six):

```python
Check =
  | FinancialMetric(metric: str, window: "annual"|"quarterly", comparator: Op, threshold: float)
  | ValuationMetric(metric: str, comparator: Op, threshold: float)
  | FactorPercentile(factor: str, cohort: str, comparator: Op, threshold: float)
  | RiskPresent(query: str, is_new: bool, min_score: float = 0.6)
  | ImpliedExpectations(field: "implied_fcf_growth"|"implied_revenue_growth",
                        comparator: Op, threshold: float)
  | PriceMetric(metric: "close"|"realized_vol"|"dist_from_high"|"max_drawdown",
                comparator: Op, threshold: float, window_days: int = 252)
```

`PriceMetric` (TA v3, 2026-09-24) is the risk-framing slice of price analysis that
[technical-analysis.md](technical-analysis.md) §2.3 admits — where a price sits against
its own history and how violent it has been. **Units are fractions**: `realized_vol` 0.35
is 35%/yr, and `dist_from_high` / `max_drawdown` are ≤ 0, so −0.25 is 25% down. `close`
alone is dollars. That breaks deliberately from the screen's percent-valued
`momentum_12_1` and from margins — the mixed convention has already produced one
silently-wrong threshold, so the newer surface picks one and says so everywhere.

**Designed, not built:**

```python
  | EventBy(query: str, by_date: date)            # events — needs parsed event dates first
  | Macro(series: str, comparator: Op, threshold: float)   # rates/energy as a condition
  | ImpliedVsGuidance(...)   # compare the market-implied bar to management's own guidance
```

`Macro` is now the closest to reachable: `wda/macro/` ships (2026-09-24) with FRED rates
and EIA energy stored point-in-time, and `get_macro_series` reads them. What is missing is
only the check type binding a series to a condition — "the 10-year stays below 5%" as a
tripwire rather than something you look up.

`RiskPresent` resolves to a boolean (present / not); `polarity` then decides whether
presence supports or breaks. The numeric checks compare a resolved value to a threshold.
Note `factor`/`metric` names and `Op` are **the same vocabularies** already exposed by
`list_factors` and `get_rule_fields`.

`EventBy` is blocked on something small: `events.event_date` is a free-text string
("Q3 FY2027", "first half of 2027"), because that is how filings state it. A parsed
earliest/latest pair alongside the text would unblock both this check and any
"catalysts in the next 30 days" question.

### 3.3 Persistence (`wda/thesis/`, migration 0008)

```
theses            (slug PK, cik, stance, horizon_months, conviction, summary,
                   valuation_anchor JSONB, status, created_at, updated_at)
thesis_conditions (id PK, thesis_slug FK, statement, polarity, severity,
                   check JSONB, hysteresis_periods)
thesis_assumptions(id PK, thesis_slug FK, label, value, provenance, note)
thesis_evaluations(id PK, thesis_slug FK, as_of, overall, detail JSONB, evaluated_at)  # history (optional)
```

The thesis is the one part of the stack that genuinely **persists** (unlike the on-demand
screen), because it is *monitored over time* — `thesis_evaluations` gives a health
timeline.

---

## 4. The MCP interface — yes, entirely through MCP

Everything — define, inspect, evaluate, monitor — is MCP tools. There is no separate UI;
the "UI" is **the claude.ai conversation plus a small, discoverable schema**. Writes are
two-step (rule 9).

### 4.1 Tools

| Tool | Kind | Purpose |
|---|---|---|
| `get_thesis_schema()` | read | the `Check` vocabulary + comparators + a worked example (the `get_rule_fields` of theses) — so the model composes valid conditions |
| `create_thesis(cik, stance, horizon_months, conviction, summary, conditions[], assumptions[], confirm)` | write (2-step) | define/replace; **preview resolves every check and returns the thesis's current health**, so you see it's well-formed *and* where it stands before saving |
| `get_thesis(slug)` | read | definition + latest evaluation |
| `evaluate_thesis(slug, as_of?)` | read | the marquee tool — the health report (§4.3) |
| `list_theses(cik?)` | read | all theses, with roll-up status |
| `update_thesis(slug, …, confirm)` / `delete_thesis(slug, confirm)` | write (2-step) | amend / retire |

### 4.2 Defining a thesis — the conversational flow

The user speaks naturally; the model, guided by `get_thesis_schema` (and the server
instructions), composes a structured `create_thesis`. A realistic transcript:

> **User:** "Open a medium-conviction long on Qualcomm, 3-year. My case: it's cheap on
> cash flow, margins are durable, and the market's underasking. Kill it if gross margin
> breaks 50 for two quarters, or a new Apple-modem risk shows up."

> **Claude →** `create_thesis(cik=804328, stance="long", horizon_months=36,
> conviction="medium", summary="Cheap FCF, durable margins, low implied bar",`
> `conditions=[`
> `  {statement:"FCF yield above 6%", polarity:"must_hold", severity:"core",`
> `   check:{type:"valuation_metric", metric:"fcf_yield", comparator:"gt", threshold:0.06}},`
> `  {statement:"Gross margin holds above 50%", polarity:"must_hold", severity:"core",`
> `   check:{type:"financial_metric", metric:"gross_margin", window:"quarterly",`
> `          comparator:"gt", threshold:50}, hysteresis_periods:2},`
> `  {statement:"No new Apple in-house modem risk", polarity:"must_not_occur",`
> `   severity:"core", check:{type:"risk_present", query:"customer in-house cellular modem",`
> `   is_new:true}},`
> `  {statement:"The price implies under 8%/yr FCF growth", polarity:"must_hold",`
> `   severity:"supporting", check:{type:"implied_expectations",`
> `   field:"implied_fcf_growth", comparator:"lt", threshold:0.08}}],`
> `confirm=false)`

Three unit conventions bite here, and all three are load-bearing:

- **`fcf_yield` is a fraction** (`0.06` = 6%), while **margins are percentages**
  (`50` = 50%). A `fcf_yield` threshold of `6` validates cleanly and then means *600%*,
  so the condition is permanently broken and never tells you anything. Rejection would
  have been kinder; this one is silent.
- **`window` is `annual` or `quarterly`**, not `latest`. The *latest period of the chosen
  window* is what gets read; `hysteresis_periods` then reaches further back.
- **`implied_expectations` compares to a number, not to guidance.** The fields are
  `implied_fcf_growth` and `implied_revenue_growth`, as annual fractions. There is no
  `vs` parameter — comparing the implied bar to management's own guidance is a good idea
  and is not built.

These examples are parsed by the validator in `tests/unit/thesis/test_doc_examples.py`,
so this block cannot drift away from the code again.

> **Claude (preview) →** *"Thesis **qcom-cheap-quality-2026** — 4 conditions, all
> resolvable. Current health: FCF yield 6.4% ✓ · gross margin 55% ✓ · no new modem risk ✓
> · implied CAGR 3pts below guidance ✓ → **intact** today. Re-run with confirm=true to
> save."*

The preview doing a **live resolve** is the key ergonomic: before it's saved you already
know the thesis is well-formed *and* see its baseline. Confirm persists it.

### 4.3 Worked examples — the newer checks

Three patterns that only became expressible in late September 2026. Each is a real
condition from a live thesis, not an illustration.

#### A price level as a tripwire

The condition [technical-analysis.md](technical-analysis.md) §2.3 named as the point of
the whole exercise. On a `watch` thesis, written per §4.4's convention as *a reason you
are still out*:

```json
{
  "statement": "The price has not yet fallen below $100 — not marked down far enough to pay me for the concentration",
  "polarity": "must_hold",
  "severity": "supporting",
  "check": {"type": "price_metric", "metric": "close", "comparator": "gt", "threshold": 100}
}
```

`intact` means still out, correctly. When it breaks, the reason to stay out has dissolved
and `changed_since_previous` names the day it happened. On a `long` the same check is the
exit tripwire instead — the polarity and the stance do the work, not a different check.

#### What a "safe" business actually did to holders

`dist_from_high` and `max_drawdown` are separate metrics because they answer different
questions. A name that fell 60% and fully recovered reads **0.0** on the first and
**−0.60** on the second.

```json
{
  "statement": "No drawdown worse than 40% in the last year — the market is not yet treating this as broken",
  "polarity": "must_hold",
  "severity": "info",
  "check": {"type": "price_metric", "metric": "max_drawdown",
            "comparator": "gt", "threshold": -0.4, "window_days": 252}
}
```

Live, this resolved to **−0.3936** on Cirrus Logic and surfaced as a *near-tripwire* — a
company screening as cheap, net-cash and stable had put holders through a 39% peak-to-trough
fall inside the year. Nothing in the fundamentals said so. `info` severity keeps it visible
without moving the roll-up, which matters on a `watch` where a falling price and a new risk
argue in opposite directions.

#### Returns on capital as a quality floor

`roe`, `roa` and `roce` joined the metric catalog on 2026-09-24, so they work in an
ordinary `financial_metric` check — no new check type needed:

```json
{
  "statement": "Return on capital employed stays above 15% for two years",
  "polarity": "must_hold",
  "severity": "core",
  "hysteresis_periods": 2,
  "check": {"type": "financial_metric", "metric": "roce", "window": "annual",
            "comparator": "gte", "threshold": 15}
}
```

**Units trap, stated once more because it has already bitten:** margins and returns are
*percentages* (`15` is 15%), `fcf_yield` and every `price_metric` fraction is a *fraction*
(`0.06` is 6%). `roce` is pre-tax — EBIT over capital employed — because an after-tax
figure would need a tax rate reconstructed from net income plus tax, a derived rate on a
derived base. A company with no long-term debt still gets a `roce`: absence of that tag
means zero, not unknown.

#### What you still cannot bind

Rates. `wda/macro/` ships and `get_macro_series` reads the 10-year, but no check type
binds a series to a condition yet, so "the 10-year stays below 5%" is something you look
up rather than something that watches itself. That is the next check to build, and the
data is already there waiting for it.

---

### 4.4 Stances — and how to write one that keeps you *out*

`long` and `short` are positions you hold. `avoid` and `watch` are the other half of the
job, and they are the more useful half before you own anything:

| Stance | What it records |
|---|---|
| `long` / `short` | a position, and the conditions that must stay true for it to remain right |
| `avoid` | a name you have looked at and rejected — **the reasons keeping you out** |
| `watch` | a name you want, at a price or on evidence you do not have yet |

**The engine is stance-blind, and that matters.** `rollup()` reads only `severity` and
`status`; it never looks at `stance`. So `intact` and `broken` always describe *the
conditions as written* — never whether that is good news for you. On a `long` the two
coincide. On a `watch` written the obvious way they invert, and a twenty-name watchlist
reports `broken` every night, which looks alarming and only means "not yet".

**The convention: write the conditions as the reasons you are still out.** Not the entry
test you are waiting to pass — the disqualifier that is currently true.

```
✗ the obvious way            "FCF yield above 5%"        must_hold
     → broken until you buy; near_tripwires silent; nightly alarm fatigue

✓ the convention             "FCF yield still below 5%"  must_hold
     → intact  = still out, correctly, nothing to do
     → broken  = the reason keeping you out is gone — go look
     → changed_since_previous names the day it dissolved
```

Written this way the third piece falls out for free. `near_tripwires` only fires on
conditions that are **currently supported and close to their bar** (`NEAR_FRACTION`, 10%),
so on an entry test it is silent forever, and on a reason-you-are-out it becomes an
approach alarm: at a `< 5%` bar, an FCF yield of 4.7% surfaces as near; 4.5% does not.
A watchlist that tells you what is getting close to interesting, without you watching.

The same shape carries `avoid`. The conditions are your disqualifiers — the price implies
more than 25%/yr, the customer-concentration risk is still in Item 1A, margins are still
falling. While they hold you stay out *with the reason on the record*, which is the part a
written-down "I passed on this" never survives. When one breaks you are told.

**The gap this routes around.** `near_tripwires` answers "what is about to break" and has
no mirror answering "what is about to be met". The convention avoids needing one. If it
ever starts to feel like a trick, the honest fix is a `near_triggers` field over broken
conditions approaching their bar, not a stance-aware roll-up — making `intact` mean
"you should act" would overload one word with two meanings and corrupt the status history
of any thesis whose stance changes.

---

### 4.5 Evaluating — the report shape

`evaluate_thesis("qcom-cheap-quality-2026", as_of="2026-11-01")` →

```json
{
  "thesis": "qcom-cheap-quality-2026", "cik": 804328, "as_of": "2026-11-01",
  "overall": "weakening",
  "conditions": [
    {"statement":"FCF yield above 6%","severity":"core","status":"supported","value":"6.1%","threshold":">6"},
    {"statement":"Gross margin holds above 50%","severity":"core","status":"weakening",
     "value":"51% (54→51 QoQ)","threshold":">50","note":"1 of 2 hysteresis quarters"},
    {"statement":"No new Apple in-house modem risk","severity":"core","status":"broken",
     "value":"NEW Item 1A risk matched (score 0.71)","tripped_at":"2026-10-30"},
    {"statement":"Market implies less growth than guidance","severity":"supporting","status":"supported","value":"-2.4pts"}
  ],
  "changed_since_inception": [
    {"condition":"No new Apple in-house modem risk","from":"supported","to":"broken"},
    {"condition":"Gross margin holds above 50%","from":"supported","to":"weakening"}
  ],
  "near_tripwires": ["Gross margin (1 more sub-50% quarter breaks it)"],
  "notes": "A core must-not-occur condition tripped on the 2026-10-30 10-Q; conviction should be revisited."
}
```

That single call — run after a new filing, or dated to any past day for a backtest — is
the whole payoff: it caught a **narrative** break (a new risk) and a **quantitative**
wobble (margin), told you *which load-bearing condition failed and when*, and flagged the
next-closest tripwire — without you re-reading the 10-Q.

### 4.6 Monitoring (server-side, surfaced via MCP)

A thesis **subscribes to its subject's filings**: the scheduler already enqueues new
filings; a small hook enqueues an `evaluate_thesis` when a subject files, and stores the
result in `thesis_evaluations`. The next `get_thesis`/`list_theses` shows the delta; an
optional alert channel can push it. No new infra — it reuses the queue + scheduler.

---

## 5. The evaluation engine

For each `Condition`: resolve its `Check` through the owning service (point-in-time via
`as_of`), compare to threshold, apply `hysteresis_periods`, and set a status —
`supported` / `weakening` (a soft breach or 1-of-N hysteresis) / `broken` / `unknown`
(data missing → honest, never a silent pass). Then a **roll-up**: any `core` condition
`broken` ⇒ thesis `broken`; any `core` `weakening` or `supporting` `broken` ⇒ `weakening`;
else `intact`. Diff against the prior evaluation (or a prior `as_of`) to produce
`changed_since`. Deterministic, rebuildable, and — like everything else — honest about
missing data.

---

## 6. Risks & challenges

- **Not everything reduces to a check.** `soft` conditions (no binding) are first-class —
  surfaced and tracked, never auto-scored; judgment still owns the call.
- **The binding is the hard part.** A badly-specified check gives false comfort. The
  curated `Check` vocabulary (vs. free-form tool bindings) is the mitigation — narrow,
  validated, model-friendly; free-form can come later behind validation.
- **Alert fatigue / flapping.** `hysteresis_periods` and `severity` exist precisely for
  this; only `core` breaks demand attention.
- **Spurious authority.** The output is a *health report*, not a buy/sell signal;
  provenance tags and explicit thresholds keep it honest.
- **Point-in-time integrity.** Backtests must date **both** facts and prices — the engine
  threads `as_of` everywhere; a mixed evaluation is a bug.

---

## 7. Phasing

- **v1 — define + evaluate, quant/narrative checks** (~2–3 days): the store + migration,
  `Check` types `financial_metric` / `valuation_metric` / `factor_percentile` /
  `risk_present`, `create_thesis` (preview resolves) + `evaluate_thesis` + `get/list`,
  provenance, point-in-time. Depends on nothing unbuilt.
- **v2 — expectational + event checks**: `implied_expectations` (needs the forward-analysis
  v1) and `event_by`; the filing-triggered monitoring hook + `thesis_evaluations` history.
- **v3 — macro checks**: `macro` type, once the macro layer lands.

---

## 8. Open questions

- Roll-up policy: is one `core` break always fatal, or weighted/scored conviction?
- Should a thesis's `valuation_anchor` be recomputed live each evaluation (re-run the
  reverse-DCF) or pinned at inception and compared?
- Alerting surface: a pull (`list_theses` shows deltas) only, or an active push channel?
- Do we let an LLM *propose* conditions from a filing, with the user approving — and how do
  we label machine-proposed vs user-authored conditions?
