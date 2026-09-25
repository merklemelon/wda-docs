# Search Corpus — Separating Retrieval from Reasoning

**Status:** proposal, 2026-09-21 (not built) — decision pending
**Related:** [ttm-fundamentals.md](ttm-fundamentals.md) §8 (failure mode 3: question/corpus
mismatch), [architecture.md](architecture.md) §L4, [decisions.md](decisions.md) (2026-09-18,
no news ingestion — the same "primary sources, broadly" instinct).

---

## 1. The proposition

Today one pipeline produces two very different things and forces them to share a coverage
policy:

| | **Reasoning enrichment** | **Search enrichment** |
|---|---|---|
| Job | `enrich.extract` (+ `enrich.embed`) | `enrich.index_filing` |
| Output | typed rows: risk themes, products, forward statements, events | passage chunks + vectors over raw prose |
| Unit cost | **$0.0458 per call**, ~$0.18 per 10-K (4 targets) | **$0.000687 per filing** |
| Value scales with | depth on a name you are studying | **breadth across names you have not** |
| Coverage today | 35 companies | 35 companies |

Indexing is **~265× cheaper per filing** than extracting. They are bundled in
`plan_enrichment` and in `arrival_chain`, so breadth is capped at whatever we are willing
to pay for depth. That is backwards.

**Why breadth is the whole point of search.** The screen narrows on numbers; semantic
search is how you narrow on *ideas* — the post-screen reduction. A corpus of 35 names can
only confirm names already chosen; it can never surface one you had not. Worse, it answers
confidently: ask "which companies flag tariff exposure" and `search_filings` returns
well-formed hits drawn from 0.7% of the market, with nothing saying so. That is failure
mode 3 in the readiness proposal, and it is **not fixable by disclosure** — only by
coverage.

---

## 2. Measured economics (2026-09-21, production)

| Measure | Value |
|---|---|
| Index cost per filing | **$0.000687** (122 runs, avg 34,370 embed tokens) |
| Extraction cost per call | **$0.0458** (737 calls) |
| Embedding of extracted rows | $0.0075 total, 643 calls |
| Chunks per 10-K | **113** |
| Current corpus | 5,792 chunks · 102 filings · 35 companies |
| Storage per chunk, all-in | **20.6 KB** (row + text + HNSW index) |
| Database today | 3,848 MB |

**Embedding spend is a rounding error.** A latest-10-K index for all 5,322 primary
tickers is ~$3.66. The cost of this proposal is *not* the model.

---

## 3. The actual constraints

### 3.1 Ingestion, not embedding

Only **68 companies have a 10-K on record at all**, and 67 have its document fetched.
1,053 companies have any filing (mostly Form 4s and 8-Ks picked up by the daily index);
1,048 have any blob. The universe is 5,322.

So a universe-wide corpus is gated on EDGAR ingestion, not on vectors:

| Step | Jobs | Notes |
|---|---|---|
| `edgar.sync_submissions` | ~5,300 | learn each company's filing history; ~30-60 s each |
| `edgar.fetch_filing` | ~5,300 | pull the primary document; 1-15 s each |
| `enrich.index_filing` | ~5,300 | chunk + embed; ~$0.0007 each |

At one serial worker that is **days of wall clock**, not hours. This is the real bill, and
it argues for a bounded, resumable backfill in the shape of the coverage reconciler rather
than one heroic run.

### 3.2 Storage is the binding limit

At 113 chunks per 10-K and 20.6 KB per chunk:

| Scope | Companies | Chunks | Added storage |
|---|---|---|---|
| Today | 35 | 5.8k | 114 MB |
| Liquid cohort (`adv_63` ≥ $5M/day) | **2,322** | ~262k | **~5.4 GB** |
| All snapshotted | 3,712 | ~419k | ~8.6 GB |
| All primary tickers | 5,322 | ~601k | ~12.4 GB |

Against a 3.8 GB database today. The full universe is a **4× database**, which is a
hosting-cost decision, not a code decision.

Two levers make this much cheaper, both available on pgvector 0.8.2 (installed, and
`halfvec` is present):

* **`halfvec(1536)`** — float16 vectors, ~half the vector bytes at negligible recall loss
  for this use. Applies to the HNSW index too.
* **Chunk size.** Fewer, larger chunks cut rows linearly. Worth measuring retrieval
  quality against, not assuming.

### 3.3 Retrieval quality at scale

We have never run this index at more than 5.8k vectors. At ~250k+, HNSW build time, `ef_search`
tuning and recall all become real questions. **A cohort-sized pilot answers them before a
universe-sized commitment**, which is the main reason to stage this.

---

## 4. Proposal

### 4.1 Separate the two tiers, in the model and in the language

Introduce **`searchable`** as a first-class tier alongside `enriched`:

* **searchable** — the filing's narrative is chunked and embedded. Cheap, wide, no LLM
  reasoning. Feeds `search_filings`.
* **enriched** — typed extraction has run. Expensive, narrow. Feeds `get_risk_themes`,
  `get_enrichment`, `search_enrichment`, thesis `risk_present`.

Concretely:

1. **Unbundle the plan.** `plan_enrichment` takes `index: bool` already; add the mirror —
   a plan that indexes *without* extracting. `arrival_chain` gains the same split so a new
   filing for a *searchable* company is indexed but not extracted.
2. **A `searchable` built-in cohort** (companies with chunks), beside `enriched`.
3. **`request_indexing(cohort|ciks)`** — a two-step write tool that plans sync → fetch →
   index for a set, with no LLM reasoning in the chain and therefore no budget-guard
   interaction beyond embeddings.
4. **A `search.reconcile_corpus` job** mirroring `enrich.reconcile_coverage`: bounded per
   night, stalest first, keeps the searchable set growing and current without being asked.

### 4.2 Scope: start with the liquid cohort, not the universe

Rank by what the user would actually act on. The `adv_63 ≥ $5M/day` cohort is **2,322
companies** — already defined, already the thing the guide calls the tradeable universe.
It is ~44% of the tickers, ~5.4 GB, and a far better answer to "which companies flag X"
than 35 names. Extend to the full universe only if retrieval holds up and the storage cost
is acceptable.

### 4.3 Tell the truth about the corpus

Whatever the scope, `search_filings` and `search_enrichment` must return a `corpus` block —
companies and filings searched, and that it is a subset — per readiness proposal §10. A
bigger corpus makes the omission *less* visible, not more, so this ships with it, not after.

---

## 5. What this does not change

* **Extraction stays narrow and deliberate.** This proposal does not widen LLM reasoning;
  it stops reasoning from gating retrieval.
* **`search_enrichment` remains the high-precision surface** over distilled rows, and stays
  at `enriched` coverage. The two searches answer different questions, and the guide's
  comparison table already says so.
* **No new sources.** Same filings, same primary-source discipline as
  [decisions.md](decisions.md) 2026-09-18.

---

## 6. Sequencing

| PR | Scope | Risk |
|---|---|---|
| 1 | Unbundle index from extract in `plan_enrichment` / `arrival_chain`; `searchable` cohort | low |
| 2 | `request_indexing` + `search.reconcile_corpus` (bounded, stalest-first) | low |
| 3 | **Pilot**: index the liquid cohort in bounded nightly batches; measure build time, `ef_search`, recall, storage | the real experiment |
| 4 | `halfvec` migration + chunk-size tuning, if the pilot says storage is binding | migration |
| 5 | `corpus` block on both search tools (or take it from readiness PR2) | low |

---

## 7. Decisions needed

1. **Scope.** Liquid cohort (2,322 / ~5.4 GB) — recommended — or all snapshotted (3,712 /
   ~8.6 GB), or the full universe (5,322 / ~12.4 GB)?
2. **Storage budget.** A 4× database changes the Supabase bill. Is that acceptable, or
   should `halfvec` and chunk tuning land *before* the pilot rather than after?
3. **Sections indexed.** All of Item 1 / 1A / 7 (today's behaviour), or Business + Risk
   only? Dropping MD&A cuts roughly a third of chunks and is the least "thematic" section.
4. **Ingestion pace.** The ~5,300 submissions syncs are days of single-worker time. Accept
   a slow bounded drain, or provision a second worker for the backfill?
5. **Does `searchable` imply prices/snapshots?** A company can be searchable without being
   rankable. Recommend keeping them independent and saying so in the tool output.
