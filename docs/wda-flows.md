# wda-flows — Agentic Pipelines That Drive WDA to Build and Monitor Theses

**Status:** proposal, 2026-09-15 (nothing built)
**Related:** [thesis.md](thesis.md) (the thesis object and tools), [forward-analysis.md](forward-analysis.md),
[fundamentals-and-valuation.md](fundamentals-and-valuation.md) (what every number means),
[gics-sectors.md](gics-sectors.md) (sector rules). The MCP server's instructions are the tool contract.

---

## 0. The idea in one paragraph

WDA is an instrument: a point-in-time, deterministic research backend that answers
questions over MCP. It does not decide what to look for. `wda-flows` is the operator — a
separate project of agentic pipelines that, given an **investment goal**, drive WDA to
select a field, write candidate **theses** from a library of **archetypes**, test them
against history, promote the survivors into a monitored library, and keep them honest as
new filings and prices arrive. The flow never touches WDA's code or database; it speaks
only the MCP contract, the same one a human uses from claude.ai. That boundary is the
whole design.

The analogy that shaped it is verification, not portfolio management. An archetype is a
testbench; a condition is an assertion; `evaluate_thesis` at a fixed `as_of` is a
regression run; an `unknown` condition is a coverage hole; the nightly diff is triage;
`confirm=true` is signoff. Nobody tunes assertions until the regression passes — that
discipline is what stops the pipeline from mining the past.

---

## 1. Vocabulary

| Term | Meaning |
|---|---|
| **Goal** | An investment objective, stated as a spec: what "good" means, over what horizon, under what constraints (e.g. *income-tilted compounders, 3-year, Industrials + Staples, no negative-FCF names*). |
| **Archetype** | A parameterized thesis definition for a *kind* of case — cheap compounder, cyclical at trough, quality growth at a fair price, turnaround. Few parameters. Lives in the flow repo as a file. |
| **Instance** | An archetype bound to one company on a date: a WDA thesis (`create_thesis`). Stored in WDA with its evaluation history. |
| **Library** | The set of saved instances WDA monitors nightly, plus the flow's index of which archetype and goal produced each. |
| **Campaign** | One run of the pipeline for one goal: field → dossiers → drafts → regression → triage → promotion. Reproducible from its manifest. |
| **Gate** | A point where a human (or an explicit policy) must confirm: promoting an instance, changing an archetype, spending LLM budget on enrichment. |
| **Objective** | The function that scores an instance's *outcome*, not its health: forward return vs its cohort over the goal's horizon, point-in-time. |

---

## 2. The boundary and the harness

**MCP is the only interface.** Every stage calls WDA tools: `screen`, `get_rule_fields`,
`list_sectors`, `get_financials`, `get_valuation`, `implied_expectations`, `get_risk_themes`,
`search_enrichment`, `get_thesis_schema`, `create_thesis`, `evaluate_thesis`, `get_thesis`,
`list_theses`, `delete_thesis`, and the `request_*` write tools behind their two-step gate.
If a stage needs something the tools don't expose, that is a WDA feature request (§8), not
an import.

**Harness.** Three options, in the order they will probably be used:

1. **Claude Code, interactively** — subagents in `.claude/agents/` with per-agent tool
   allowlists and models, a workflow script for the fan-out stages. The fastest way to
   prototype the agent prompts and the archetypes.
2. **Claude Agent SDK** — the same harness as a Python program: the orchestration becomes
   code (stages, gates, retries, budgets, manifests), the subagent definitions carry over
   unchanged, the WDA MCP server is attached by URL. This is the unattended, scheduled form.
3. **Managed Agents** — if hosting and scheduling should move off the workstation.

All three share one property that matters here: the subagent definitions and skills are
files, so the flow's *knowledge* is versioned in the repo regardless of where it runs.

**Repo layout (proposed):**

```
wda-flows/
  CLAUDE.md                 how the flow talks to WDA; the gate policy; the anti-mining rules
  .claude/agents/           one file per role (§4): scout, analyst, author, skeptic, librarian,
                            judge, sentinel, reviewer
  .claude/skills/           procedures: instantiate-archetype, run-regression, triage-diff, ...
  archetypes/               *.yaml — the parameterized thesis definitions (§5)
  goals/                    *.yaml — investment goal specs (§3.0)
  flows/                    orchestration: workflow scripts now, SDK programs later
  eval/                     fixed as_of dates, the held-out set, the objective implementation
  library/index.yaml        which archetype + goal + campaign produced each saved thesis slug
  runs/<campaign-id>/       manifest.json, dossiers/, drafts/, regression.csv, triage.md
```

---

## 3. The pipeline

Each stage is a subagent with a narrow tool allowlist, reading and writing files under
`runs/<campaign>/`. Stages are idempotent and resumable: a campaign can be re-run from any
stage using the manifest, and every WDA read carries the campaign's `as_of` so a re-run
sees the same world.

### 3.0 Goal intake

The goal spec is the campaign's contract:

```yaml
# goals/income-compounders-industrials.yaml
name: income-compounders-industrials
objective: total return vs cohort over horizon, with drawdown penalty
horizon_months: 36
universe:
  base: screenable
  rules:
    - [gics_sector, in, [Industrials, Consumer Staples]]
    - [fcf_yield, gt, 0]
    - [coverage, gte, 8]            # a proposed WDA rule field, see §8
archetypes: [cheap-compounder, quality-at-fair-price]
field_size: 40                     # candidates to dossier
budget:
  enrichment_usd: 25               # LLM spend the campaign may request
  max_new_theses: 8
gates: [promote, enrich]
```

A tiny agent (or none — this can be a form) validates the spec against `get_rule_fields`
and `list_sectors` so a goal never references a field or sector that doesn't exist.

### 3.1 Field selection — the *scout*

Tools: `screen`, `get_rule_fields`, `list_sectors`, `list_cohorts`, `get_score`.

The scout turns the goal into a cohort rule, screens it, and produces the **field**: the
top `field_size` names plus a coverage report — how many of the twelve factors each
candidate has, which are missing, whether it is enriched. It writes `field.csv` and a
short note on what the screen is and isn't measuring (the snapshot's `basis` and
`price_date` go into the manifest). It does *not* judge companies; it selects a field the
later stages can afford.

### 3.2 Dossier — the *analyst*

Tools: `get_company`, `get_financials`, `get_valuation`, `implied_expectations`,
`get_risk_themes`, `get_enrichment`, `search_enrichment`, `list_filings`.

One dossier per candidate, structured (not prose): the last 8 fiscal years and 8
quarters of the core metrics, the multiples, the reverse-DCF band with its three bases and
cyclical flag, and — where the company is enriched — the new risks, products and forward
statements. The analyst records *gaps* explicitly ("not enriched", "no gross margin
tagged", "bank: no revenue concept") because the author must not write conditions the
data cannot check. If enrichment is missing and the goal allows it, the analyst files an
**enrichment request** (`request_fetch` → `request_enrichment` previews) against the
campaign's budget; the `enrich` gate decides.

### 3.3 Drafting — the *author* and the *skeptic*

Tools: `get_thesis_schema`, `create_thesis` (preview only — the author is not allowed
`confirm=true`).

The **author** instantiates each applicable archetype for each candidate: it fills the
archetype's parameters from the dossier (a compounder archetype needs the company's own
margin history to set its floor, not a global number), writes the human statements, and
binds each to a check from the schema. It calls `create_thesis(confirm=false)` — the
preview resolves every check live and returns the baseline health, so a malformed or
unresolvable draft is caught immediately. Drafts whose *core* conditions are `unknown` at
inception are rejected: a thesis you cannot check on day one is not a thesis.

The **skeptic** then reads each draft with one job: add the tripwires the author didn't.
Concentration risk, the customer that could insource, the margin that has never survived a
downturn, the implied growth that exceeds anything the company has ever delivered. It
proposes `must_not_occur` conditions — bound where a check exists (`risk_present`,
`financial_metric` with hysteresis), *soft* where it doesn't — and it may lower a
condition's severity but never raise the thesis's conviction. The two agents disagree by
design; the draft that survives is the pair's joint work.

### 3.4 Regression — the *judge*

Tools: `evaluate_thesis` (with `as_of`), `get_valuation`, `get_prices`.

For every draft, the judge runs `evaluate_thesis(as_of=d)` at the campaign's **fixed
historical dates** — say the first trading day of each of the last eight quarters — and
records the health at each date. Then it scores the *outcome*: what did the stock do over
the goal's horizon from each date where the thesis was intact, relative to its cohort?
That needs a point-in-time forward-return read (§8, the one WDA feature the pipeline
cannot do without).

Two rules make this a regression rather than a fishing trip:

- **Held-out dates.** A subset of dates (say the two most recent quarters) is never used
  to choose among drafts or to tune archetypes. It is reported, not optimized.
- **Archetypes are scored, instances are ranked.** The question "is this archetype any
  good?" is answered over *all* its instances across campaigns, in `eval/`. A single
  instance's backtest ranks it within the field; it never edits the archetype.

Output: `regression.csv` — one row per (draft, date): health, outcome, cohort outcome.

### 3.5 Triage and promotion — the *librarian* and the *reviewer* gate

Tools: `list_theses`, `get_thesis`, `create_thesis` (the librarian is the only agent
allowed `confirm=true`), `delete_thesis`.

The librarian ranks the survivors: intact at inception, intact at most historical dates,
outcome above cohort on the in-sample dates, held-out reported honestly. It de-duplicates
against the existing library (`list_theses(cik)` — one company, one active thesis per
archetype), enforces `max_new_theses`, and produces `triage.md`: the shortlist with, for
each, the archetype, the baseline health, the regression summary, and the skeptic's
tripwires in plain English.

The **promote gate** is human: you read `triage.md` and say which to save. The librarian
then calls `create_thesis(confirm=true)` and appends to `library/index.yaml`. In an
unattended SDK run the gate can be a policy ("auto-promote if intact on ≥ 6 of 8 dates and
outcome > cohort on all in-sample dates"), but the default is a person.

### 3.6 Monitoring — the *sentinel*

Tools: `list_theses`, `get_thesis`, `evaluate_thesis`.

WDA already re-evaluates every saved thesis nightly (20:00 ET) and records the diff. The
sentinel runs each morning: reads `list_theses`, pulls `changed_since_previous` for anything
that moved, and routes:

- a **core** break → an alert with the condition, its value, the threshold, and the
  filing/price that moved it; the thesis is flagged for review, not auto-retired;
- a **weakening** → a watch note; near-tripwires get listed;
- an **unknown** that used to resolve → a data alert (a concept changed, a price is
  missing) routed to WDA maintenance, not to the investor.

Retirement is a human gate too (`delete_thesis` two-step), but the sentinel drafts the
case: horizon elapsed, thesis broken for N consecutive evaluations, or the goal retired.

### 3.7 The learning loop — the *reviewer*

Quarterly, not nightly. The reviewer reads `eval/` across campaigns: per archetype, how
often instances were intact, how outcomes compared to cohorts, which check types resolved
and which sat `unknown` (coverage), which tripwires actually fired. It writes a
**thesis review board** report and proposes archetype changes as diffs — parameters,
conditions, severities — for you to accept. Changes are small, dated, and logged in the
archetype file's changelog. Nothing tunes itself.

---

## 4. The agent roster

| Agent | Model | Tools (allowlist) | Writes | Never |
|---|---|---|---|---|
| scout | Sonnet | screen, get_rule_fields, list_sectors, list_cohorts, get_score | `field.csv` | judges a company |
| analyst | Sonnet | get_company, get_financials, get_valuation, implied_expectations, get_risk_themes, get_enrichment, search_enrichment, request_fetch/enrichment (preview) | `dossiers/` | writes a condition |
| author | Opus | get_thesis_schema, create_thesis (preview) | `drafts/` | `confirm=true` |
| skeptic | Opus | get_thesis_schema, search_enrichment, get_risk_themes | `drafts/` (tripwires) | raises conviction |
| judge | Sonnet | evaluate_thesis, get_valuation, get_prices, (forward-outcome) | `regression.csv` | edits an archetype |
| librarian | Sonnet | list_theses, get_thesis, create_thesis (confirm), delete_thesis | `triage.md`, `library/index.yaml` | promotes without the gate |
| sentinel | Haiku | list_theses, get_thesis, evaluate_thesis | alerts | retires a thesis |
| reviewer | Opus | files only (eval/, archetypes/) | review report, archetype diffs | applies its own diffs |

Model choices are a starting point: the judgment-heavy roles (author, skeptic, reviewer)
get the strongest model; the mechanical ones (scout, judge, sentinel) don't need it. All
Claude, per preference. The allowlists are the safety model — an agent cannot call a tool
it doesn't have, so "the author cannot save" is enforced by the harness, not by prompt.

---

## 5. Archetypes

An archetype is a YAML file: parameters with allowed ranges, how each is derived from the
dossier, the conditions as templates over the WDA check vocabulary, and a changelog.

```yaml
# archetypes/cheap-compounder.yaml
name: cheap-compounder
intent: >
  A business whose cash generation is durable and whose price asks little of it.
  Wins when the market re-rates or simply compounds at the underlying FCF growth.
stance: long
horizon_months: 36
parameters:
  fcf_yield_floor:      {default: 0.05, range: [0.04, 0.08]}
  margin_floor:         {derive: "0.9 x median(gross_margin, last 5 FY)", range: [30, 70]}
  margin_hysteresis:    {default: 2}
  implied_growth_cap:   {derive: "min(0.10, revenue_cagr_5y)"}
conditions:
  - statement: "FCF yield above {fcf_yield_floor:.0%}"
    severity: core
    check: {type: valuation_metric, metric: fcf_yield, comparator: gt, threshold: "{fcf_yield_floor}"}
  - statement: "Gross margin holds above {margin_floor:.0f}% for {margin_hysteresis} periods"
    severity: core
    hysteresis_periods: "{margin_hysteresis}"
    check: {type: financial_metric, metric: gross_margin, window: quarterly, comparator: gt, threshold: "{margin_floor}"}
  - statement: "Market implies less growth than the company has delivered"
    severity: supporting
    check: {type: implied_expectations, field: implied_fcf_growth, comparator: lt, threshold: "{implied_growth_cap}"}
  - statement: "Debt stays serviceable"
    severity: supporting
    check: {type: financial_metric, metric: debt_to_equity, comparator: lt, threshold: 1.5}
tripwire_prompts:            # what the skeptic looks for on this archetype
  - customer or supplier concentration that could reset the margin
  - a capital-allocation change (a large acquisition, a dividend cut)
changelog:
  - 2026-09-15: created
```

Design rules for archetypes: **few parameters**, each with a *derivation* from the
company's own history where possible (a margin floor relative to its own median, not a
global 50), and a written **intent** that a human can disagree with. The intent is what
the reviewer checks proposals against.

---

## 6. The objective and the anti-mining rules

The pipeline's power is also its hazard: with `as_of` it can evaluate any definition at
any past date, and a search over enough definitions will find one that "worked." The
rules that keep it honest are procedural and simple, and they belong in the flow's
`CLAUDE.md` so every agent reads them:

1. **Fixed dates, chosen once.** A campaign's evaluation dates are set at intake and
   recorded in the manifest; no stage adds dates.
2. **A held-out set that is reported, never optimized.** The most recent quarters are
   for reporting only.
3. **Instances are ranked; archetypes are scored** — over all their instances, across
   campaigns, quarterly, by the reviewer, with diffs a human accepts.
4. **Outcome vs cohort, not vs zero.** A thesis that made 20% in a market that made 30%
   did not work. The cohort is the screen's own field for that goal.
5. **Depth honesty.** WDA's price history and five-year facts window limit how far back
   a regression can reach; the manifest records the earliest usable date, and results
   past it are marked as such rather than silently thin.
6. **Survivorship.** The universe is today's filers; a 2022 date sees today's companies,
   not 2022's. Reported as a known bias; not fixable without historical universes.

---

## 7. A campaign, end to end

*Goal:* income-tilted compounders in Industrials and Consumer Staples, three years.

1. **Scout** — `screen(rule={base: screenable, clauses: [[gics_sector, in, [Industrials, Consumer Staples]], [fcf_yield, gt, 0]]}, top_k=40)`; field of 40 with coverage ≥ 8 of 12 factors; notes the snapshot's `price_date`.
2. **Analyst** — 40 dossiers; 31 have a full margin history, 6 are not enriched (no `risk_present` checks possible — recorded), 3 are excluded for negative FCF.
3. **Author** — instantiates `cheap-compounder` and `quality-at-fair-price` where applicable: 52 drafts; 9 rejected at preview for `unknown` core conditions.
4. **Skeptic** — adds tripwires to the 43: for a defense supplier, "no new customer-concentration risk" bound to `risk_present(is_new=true)`; for a distributor, "operating margin never below 4% for two quarters".
5. **Judge** — `evaluate_thesis(as_of=…)` at 8 quarterly dates × 43 drafts; forward outcomes vs the field; held-out = the last 2 dates.
6. **Librarian** — shortlist of 8 within the goal's cap; `triage.md` with each thesis's story in plain English.
7. **You** — promote 5. The librarian saves them; `library/index.yaml` gains five rows tagged with the goal, archetype, campaign id.
8. **Sentinel** — from the next morning: nothing moved. Six weeks later: one core tripwire fires on a 10-Q (a new Item 1A risk matched the concentration query at 0.71) — an alert with the filing link and the sentence that matched.
9. **Reviewer** — next quarter: across three campaigns, `cheap-compounder` instances were intact 78% of the time; the implied-growth condition never fired; the margin hysteresis of 2 caught two false alarms that 1 would have tripped. Proposal: keep. `quality-at-fair-price` sat `unknown` on its R&D-intensity condition for 40% of instances — the reviewer proposes dropping it to *supporting*. You accept.

---

## 8. What WDA needs to add (small, in order)

| Need | Why | Size |
|---|---|---|
| **Forward-outcome tool** — `get_forward_return(cik, as_of, horizon_months, cohort?)`: total return from `as_of` over the horizon, point-in-time, vs the cohort's median | the objective function; nothing else in the pipeline is blocked | ~½ day |
| **Batch evaluation** — `evaluate_theses(names, as_of)` or a matrix over dates | 43 drafts × 8 dates = 344 calls today | ~½ day |
| **`coverage` as a rule field** | so a goal can require ≥ N factors without post-filtering | hours |
| **`event_by` check** (thesis v2) | catalysts with a date | ~½ day |
| **MCP endpoint config for the SDK** (auth, URL) | the unattended harness | hours, once |
| **Thesis templates in WDA** | only after 2–3 archetypes prove themselves; until then they are files in the flow repo | deferred |

Also the two items already queued independently of flows: the **scheduled facts refresh**
(so instances' fundamentals move when companies file) and the **coverage floor** on the
universe screen.

---

## 9. Risks, and what the design does about them

- **Data mining** — §6. The pipeline makes it easy to overfit; the rules make it hard to
  do so quietly.
- **Sparse narrative data** — only ~22 companies are enriched. Narrative checks are the
  most interesting tripwires and the least available. The `enrich` gate and budget make
  enrichment a deliberate campaign cost, prioritized for the shortlist, not the field.
- **LLM nondeterminism in drafting** — the author's *choices* vary run to run; the
  checks it emits do not (they are validated against a closed vocabulary), and the
  regression is deterministic. Drafts are artifacts in `runs/`, diffable across runs.
- **Alert fatigue** — severity + hysteresis in WDA, plus the sentinel's routing: only
  core breaks reach you; weakening is a list you read when you choose.
- **Spurious authority** — every output is a health report or a ranking with the evidence
  attached. The gates keep the decision human. The pipeline recommends; it does not trade.
- **Cost** — WDA reads are cheap (the screen is milliseconds off the snapshot);
  enrichment and the strong-model agents are the spend, both capped per campaign.

---

## 10. Phasing

- **v0 — one archetype, one goal, by hand in Claude Code** (days): write
  `cheap-compounder.yaml`, the scout/analyst/author/skeptic subagents, run a campaign
  interactively over a 20-name field, promote two theses manually. Learn what the
  agents get wrong.
- **v1 — the regression and the objective** (a week, plus the WDA forward-outcome tool):
  fixed dates, held-out set, `regression.csv`, the librarian's triage, the promote gate.
- **v2 — unattended** (SDK): campaigns as programs, the sentinel on a morning schedule,
  manifests, resume from any stage.
- **v3 — the learning loop**: the reviewer, archetype scoring across campaigns, the
  review-board report, changelogs.

The first thing to build is not an agent. It is `archetypes/cheap-compounder.yaml` and
`goals/<one goal>.yaml`, written by you — the pipeline exists to instantiate, test and
monitor *your* definitions, not to invent them.
