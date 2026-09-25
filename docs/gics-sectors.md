# GICS Sector Classification — Design

**Status:** proposed (not built)
**Author:** design note, 2026-09-14
**Related:** `wda/scoring/rules.py` (the `sector` rule field), `wda/scoring/cohorts.py`
(`factor_rows`), `wda/universe/` (company/SIC storage).

> **Status (2026-09-15): implemented** — `wda/universe/gics.py`, the `gics_sector` rule
> field, `list_sectors`, and the `submissions.zip` SIC backfill (`bulk_submissions_cli`).
> §4 remains the source of truth for the crosswalk; the code mirrors it table-for-table.

## 1. Motivation

Cohort rules already filter on `sector`, which is the SEC's **SIC**
(`sic_description`) — a granular, 1930s-era taxonomy (e.g. *"Semiconductors & Related
Devices"*, *"Pharmaceutical Preparations"*). Many investors and financial planners
instead think in the **11 GICS sectors** (S&P/MSCI): Information Technology, Health
Care, Financials, Consumer Discretionary, Communication Services, Industrials, Consumer
Staples, Energy, Utilities, Real Estate, Materials.

Goal: expose a **`gics_sector`** rule field alongside `sector`, so a rule can say
`["gics_sector", "eq", "Information Technology"]`, while keeping the precise SIC filter
for those who want it.

## 2. Current state — the blocker

`sic`/`sic_description` is populated for **only 24 of 5,318 companies** (10 distinct SIC
descriptions). The rest arrived via the bulk `companyfacts.zip` backfill and the ticker
sync, neither of which carries SIC. So **today `sector` (and any `gics_sector`) filters
match only those ~24 curated names** — not the 1,154-company screenable universe.

**Prerequisite:** backfill `sic`/`sic_description` across the universe before GICS
filtering is broadly useful. Options, cheapest first:

- **SEC bulk `submissions.zip`** (data.sec.gov) — one download; each company's JSON
  carries `sic` and `sicDescription`. Mirrors the existing `bulk_companyfacts_cli`
  pattern (one-off CLI, not a queue job). **Recommended.**
- **Per-company `edgar.sync_submissions`** — already parses SIC, but running it for the
  whole universe is heavy and was the source of the poison-job worker hang.
- Do nothing and accept sector filtering only on the enriched/curated set.

This backfill is a separate task; the GICS mapping below is worthwhile regardless (it
turns whatever SIC coverage exists into GICS), but its reach equals SIC coverage.

## 3. Proposed design

### 3.1 Map on the numeric SIC code, not the description

`Company` stores both `sic` (4-digit integer) and `sic_description` (text). Map on the
**numeric code** — it's stable, and SIC is hierarchical: the 2-digit **major group**
(e.g. `36` = Electronic & Other Electrical Equipment) is the natural mapping unit, with
a handful of 4-digit **overrides** where a major group splits across GICS (pharma,
computers, autos, REITs…). String-matching descriptions is brittle and misses the
hierarchy.

### 3.2 A pure mapping module — `wda/universe/gics.py`

```python
GICS_SECTORS: tuple[str, ...] = (...11 names...)

def gics_for_sic(sic: int | None) -> str | None:
    """Map a 4-digit SIC code to one of the 11 GICS sectors (best-effort).
    Checks 4-digit overrides first, then the 2-digit major group."""
```

- `_OVERRIDES: dict[int, str]` — specific 4-digit codes (see §4.2).
- `_MAJOR_GROUPS: dict[int, str]` — 2-digit prefix → GICS (see §4.1).
- Pure, table-driven, fully unit-testable; no DB, no network. Lives in `universe`
  (unconstrained), so `scoring` may import it.

### 3.3 Wire `gics_sector` into rules

- `wda/scoring/rules.py`: add `gics_sector` to `STRING_FIELDS`.
- `wda/scoring/cohorts.py` `SqlCohortSources.factor_rows`: fetch each company's `sic`
  (extend `SqlCompanyRepository.sectors_for` → also return codes, or add
  `sic_codes_for`), derive `gics_sector = gics_for_sic(sic)`, and add it to the row
  alongside `sector`. Companies with no SIC get `gics_sector=None` (a rule clause on a
  `None` field never matches — same rule as any missing factor).

### 3.4 Discoverability — `list_sectors` tool

New read tool returning the distinct `sic_description` **and** `gics_sector` values
actually present in the corpus, with counts:

```
list_sectors() -> {"sectors": [{"sic_description", "gics_sector", "count"}, ...],
                   "gics_totals": {"Information Technology": N, ...}}
```

So the model (or user) filters on real values instead of guessing. `get_rule_fields`
gains `gics_sector` in its field list.

### 3.5 Tests

- `gics_for_sic`: overrides beat major groups; a few representative codes per GICS
  sector; `None` and unknown codes → `None`.
- rule evaluation with a `gics_sector` clause (unit, fake rows).
- integration: a `gics_sector` rule cohort over seeded companies with SIC codes.

### 3.6 Non-goals / caveats

SIC predates modern industry structure, so the translation is **approximate** — most
visibly for **Communication Services** (GICS moved big internet/media here; SIC scatters
them across *Services-Computer* `737x` and *Communications* `48xx`). A SIC→GICS map
cannot perfectly reproduce index-provider classifications (which also use revenue mix and
judgment). This is a good-enough bridge, clearly labelled best-effort — not an official
GICS assignment. A later refinement could map on ticker via a curated GICS list for
S&P/index members, falling back to SIC for the rest.

## 4. SIC → GICS crosswalk

### 4.1 By SIC major group (2-digit) — the default mapping

| SIC | Major group | GICS sector |
|----:|-------------|-------------|
| 01 | Agricultural Production — Crops | Consumer Staples |
| 02 | Agricultural Production — Livestock | Consumer Staples |
| 07 | Agricultural Services | Consumer Staples |
| 08 | Forestry | Materials |
| 09 | Fishing, Hunting & Trapping | Consumer Staples |
| 10 | Metal Mining | Materials |
| 12 | Coal Mining | Energy |
| 13 | Oil & Gas Extraction | Energy |
| 14 | Nonmetallic Minerals Mining | Materials |
| 15 | Building Construction — General Contractors | Consumer Discretionary¹ |
| 16 | Heavy Construction (non-building) | Industrials |
| 17 | Construction — Special Trade Contractors | Industrials |
| 20 | Food & Kindred Products | Consumer Staples |
| 21 | Tobacco Products | Consumer Staples |
| 22 | Textile Mill Products | Consumer Discretionary |
| 23 | Apparel & Finished Fabric Products | Consumer Discretionary |
| 24 | Lumber & Wood Products | Materials |
| 25 | Furniture & Fixtures | Consumer Discretionary |
| 26 | Paper & Allied Products | Materials |
| 27 | Printing & Publishing | Communication Services² |
| 28 | Chemicals & Allied Products | Materials³ |
| 29 | Petroleum Refining & Related | Energy |
| 30 | Rubber & Misc. Plastics | Materials |
| 31 | Leather & Leather Products | Consumer Discretionary |
| 32 | Stone, Clay, Glass & Concrete | Materials |
| 33 | Primary Metal Industries | Materials |
| 34 | Fabricated Metal Products | Industrials |
| 35 | Industrial/Commercial Machinery & Computer Equipment | Industrials⁴ |
| 36 | Electronic & Other Electrical Equipment | Information Technology⁵ |
| 37 | Transportation Equipment | Industrials⁶ |
| 38 | Measuring/Analyzing/Controlling Instruments | Health Care⁷ |
| 39 | Miscellaneous Manufacturing | Consumer Discretionary |
| 40 | Railroad Transportation | Industrials |
| 41 | Local & Suburban Transit | Industrials |
| 42 | Motor Freight Transportation & Warehousing | Industrials |
| 44 | Water Transportation | Industrials |
| 45 | Transportation by Air | Industrials |
| 46 | Pipelines (except Natural Gas) | Energy |
| 47 | Transportation Services | Industrials |
| 48 | Communications | Communication Services⁸ |
| 49 | Electric, Gas & Sanitary Services | Utilities⁹ |
| 50 | Wholesale Trade — Durable Goods | Industrials¹⁰ |
| 51 | Wholesale Trade — Nondurable Goods | Consumer Staples |
| 52 | Building Materials & Garden Supplies (retail) | Consumer Discretionary |
| 53 | General Merchandise Stores | Consumer Discretionary |
| 54 | Food Stores | Consumer Staples |
| 55 | Automotive Dealers & Service Stations | Consumer Discretionary |
| 56 | Apparel & Accessory Stores | Consumer Discretionary |
| 57 | Home Furniture & Furnishings Stores | Consumer Discretionary |
| 58 | Eating & Drinking Places | Consumer Discretionary |
| 59 | Miscellaneous Retail | Consumer Discretionary¹¹ |
| 60 | Depository Institutions (banks) | Financials |
| 61 | Nondepository Credit Institutions | Financials |
| 62 | Security & Commodity Brokers | Financials |
| 63 | Insurance Carriers | Financials |
| 64 | Insurance Agents & Brokers | Financials |
| 65 | Real Estate | Real Estate |
| 67 | Holding & Other Investment Offices | Financials¹² |
| 70 | Hotels & Other Lodging | Consumer Discretionary |
| 72 | Personal Services | Consumer Discretionary |
| 73 | Business Services | Information Technology¹³ |
| 75 | Automotive Repair & Services | Consumer Discretionary |
| 76 | Miscellaneous Repair Services | Industrials |
| 78 | Motion Pictures | Communication Services |
| 79 | Amusement & Recreation Services | Communication Services¹⁴ |
| 80 | Health Services | Health Care |
| 81 | Legal Services | Industrials |
| 82 | Educational Services | Consumer Discretionary |
| 83 | Social Services | Health Care |
| 87 | Engineering, Accounting, Research & Mgmt | Industrials |
| 89 | Miscellaneous Services | Industrials |
| 99 | Nonclassifiable Establishments | *(unmapped)* |

### 4.2 Notable 4-digit overrides (checked before the major group)

| SIC | Description | Major-group default | GICS override |
|----:|-------------|--------------------|---------------|
| 2833–2836 | Medicinal, Pharmaceutical & Biological Products | Materials | **Health Care** |
| 3570–3579 | Computer & Office Equipment | Industrials | **Information Technology** |
| 3661–3669 | Telephone & Telegraph / Comm. Equipment | (IT) | **Information Technology** |
| 3674 | Semiconductors & Related Devices | (IT) | **Information Technology** |
| 3690–3699 | Misc. Electrical Machinery & Supplies | (IT) | Industrials⁵ |
| 3711–3716 | Motor Vehicles & Car Bodies | Industrials | **Consumer Discretionary** |
| 3721–3728 | Aircraft & Parts | Industrials | Industrials (Aerospace & Defense) |
| 3826/3829 | Laboratory & Measuring Instruments | (HC) | Health Care / Information Technology |
| 3841–3851 | Surgical, Medical & Dental Instruments | (HC) | **Health Care** |
| 4813–4899 | Telecom services / Broadcasting | (Comm) | **Communication Services** |
| 4911–4991 | Electric/Gas/Water Utilities | Utilities | Utilities |
| 5912 | Drug Stores & Proprietary Stores | (Retail) | Consumer Staples |
| 6021–6199 | Banks & Credit | Financials | Financials |
| 6311–6411 | Insurance | Financials | Financials |
| 6500–6599 | Real Estate | Real Estate | Real Estate |
| 6770/6798 | Blank Checks / **REITs** | Financials | **Real Estate** |
| 7311 | Advertising | (IT/Bus. Svcs) | Communication Services |
| 7370–7379 | Computer Programming, Software, Data Processing | (IT) | **Information Technology**⁸ |
| 7812–7900 | Motion Picture & Entertainment | Services | Communication Services |
| 8000–8099 | Health Services | Health Care | Health Care |

### 4.3 SIC descriptions currently in the corpus (24 companies)

| Count | SIC description | SIC | GICS (proposed) |
|------:|-----------------|----:|-----------------|
| 11 | Semiconductors & Related Devices | 3674 | Information Technology |
| 3 | Services-Prepackaged Software | 7372 | Information Technology |
| 2 | Electronic Computers | 3571 | Information Technology |
| 2 | Services-Computer Programming, Data Processing, Etc. | 7389 | Information Technology |
| 1 | Communications Equipment, NEC | 3669 | Information Technology |
| 1 | Fire, Marine & Casualty Insurance | 6331 | Financials |
| 1 | Motor Vehicles & Passenger Car Bodies | 3711 | Consumer Discretionary |
| 1 | Radio & TV Broadcasting & Communications Equipment | 3663 | Information Technology |
| 1 | Retail-Catalog & Mail-Order Houses | 5961 | Consumer Discretionary |
| 1 | Wholesale-Electronic Parts & Equipment, NEC | 5065 | Information Technology¹⁵ |

## 5. Footnotes (judgment calls)

1. **15** Homebuilders sit in Consumer Discretionary under GICS; general/heavy
   *contractors* are Industrials. Split at the 4-digit level if precision matters
   (152x homebuilding → Consumer Discretionary; 162x → Industrials).
2. **27** Publishing → Communication Services (Media). Commercial printing alone is
   closer to Industrials.
3. **28** Chemicals default to Materials; **pharma/biotech (283x) → Health Care** via
   the override.
4. **35** Machinery is Industrials, but **computer equipment (357x) → IT** via override.
5. **36** Electronic/electrical equipment leans IT (semis, comm equipment); heavy
   *electrical* apparatus (industrial motors, 3690s) is arguably Industrials.
6. **37** Transportation equipment is Industrials, but **autos (371x) → Consumer
   Discretionary**; aerospace & defense stays Industrials.
7. **38** Instruments split: **medical (384x) → Health Care**; industrial/scientific
   measurement can be IT or Industrials.
8. **48 / 737x** The hardest area: GICS Communication Services blends telecom + media +
   interactive/internet. SIC has no "internet" class — internet firms are usually
   `737x` (which we map to IT). Big platform names (Alphabet, Meta) are Communication
   Services under GICS but IT under a pure SIC map — a known, documented gap.
9. **49** Utilities; sanitary/waste services (4950s) are arguably Industrials
   (Environmental Services) under GICS.
10. **50/51** Distributors: GICS spreads these across Industrials (Trading Companies &
    Distributors), Consumer, and IT (Technology Distributors, e.g. Avnet). Default 50 →
    Industrials, but electronic-parts wholesale (5045/5065) → IT.
11. **59** Misc retail defaults to Consumer Discretionary; drug stores (5912) → Consumer
    Staples.
12. **67** Holding/investment offices → Financials, but **REITs (6798) → Real Estate**.
13. **73** Business services default to IT (dominated by software/data `737x`);
    advertising (7311) → Communication Services; staffing/facilities → Industrials.
14. **79** Recreation → Communication Services (Entertainment) for GICS; some (casinos,
    resorts) are Consumer Discretionary.
15. **5065** Electronic-parts distribution → IT (Technology Hardware / Distributors),
    matching GICS' treatment of distributors like Avnet.

## 6. Effort & phasing

- **Feature (mapping + `gics_sector` field + `list_sectors`):** ~½ day. Self-contained,
  no migration; ships as one PR with tests + deploy.
- **Prerequisite (universe SIC backfill via `submissions.zip`):** ~½–1 day; its own PR.
  Without it the feature works but only over the ~24 SIC-bearing names.
- Recommended order: backfill SIC first (so the feature has data to classify), then the
  mapping + field. They're independent, so either order works.
