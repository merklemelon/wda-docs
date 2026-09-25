# Decisions (ADR-style)

One entry per decision: date, decision, alternatives, reason. Newest first.

---

## 2026-09-18 — No news ingestion; primary sources only
**Decision:** WDA does not ingest news. The v0.4 vision document
([reference/worlddomination-architecture.md](https://github.com/merklemelon/wda) (the v0.4 vision document lives in the private app repo))
sketched an L0 news fetcher, L4 news linking and a `news` table; none was built and none
will be for now. Narrative input comes from primary records only: SEC filings (10-K,
10-Q, 8-K and their press-release exhibits) and, per the macro plan, the Federal
Register, FRED and EIA.

**Alternatives considered:** a news API (Tiingo bundles one) for event detection and
product/company linking; curating a set of "trusted" outlets.

**Reason:**
- **We already have the source text.** An 8-K and its Exhibit 99 press release are the
  primary documents wire stories are written from, and since 2026-09-17 they are fetched
  and extracted on arrival — same hour, no intermediary.
- **Framing risk.** Any outlet introduces characterization. Curating "trusted" outlets
  would bake one party's editorial judgment into a research system, and it is not
  auditable by whoever later reads a score or a thesis. Depending on interpretation as
  little as possible is the stronger guarantee than trying to source unbiased
  interpretation.
- **Cost/benefit.** Entity linking (deciding an article is about *that* product from
  *that* filing) is fuzzy and was rated Medium difficulty in the vision doc. It buys
  little over the 8-K path while adding a tuning burden.

**If revisited**, the constraints are: news may only *detect events*, never interpret
them; every claim must carry outlet, timestamp and the primary record it points to;
narrative alone must never move a factor or trip a thesis condition. Wire services
(Reuters, AP, Dow Jones) are structurally preferable because the same feed is sold to
every desk, so the commercial incentive favours accuracy over audience capture.

---

## 2026-09-10 — fly.io web deploy (app `wda`, public MCP)
**Decisions made deploying stories 2.2/2.3:**
- **Stateless MCP** (`streamable_http_app(stateless_http=True)`) — the transport
  is otherwise per-session; behind ≥2 fly machines a follow-up call round-robins
  to an instance without the session → "Session not found". Stateless removes
  server-side session state (our tools are stateless).
- **Host guard** — the transport's DNS-rebinding protection allows only
  localhost by default → 421 "Invalid Host header" for `wda.fly.dev`. Added
  `WDA_MCP_ALLOWED_HOSTS` (empty=SDK default, `*`=off, else allowlist); set to
  `wda.fly.dev` on fly. Acceptable since the endpoint is bearer-token + HTTPS.
- **Public IPs** — fly's auto-allocation failed on first deploy
  (`org_slug is only supported with private_v6`); allocated manually
  (`fly ips allocate-v4 --shared` + `allocate-v6`).
- **Migrations** run via fly `release_command = alembic upgrade head` (once per
  deploy), not on machine boot. Image built by fly's remote builder (local Docker
  was down). New app `wda` in the personal org — isolated from `btc-explorer`.

## 2026-09-10 — Production DB: dedicated Supabase project `wda`
**Decision:** Created a new, isolated Supabase project **`wda`** (ref
`dxeiixrgyqxwttpgetop`, region us-east-1, Postgres 17.6) — separate from the
existing `MerkleMelon` project. Migration 0001 applied cleanly (7 tables, 17
fy-partitions, pgvector enabled). Connect via the **Supavisor pooler**: session
mode (`:5432`) for worker/scheduler, transaction mode (`:6543`) for web/MCP.
**Alternatives:** reuse `MerkleMelon` (rejected — isolation); direct connection
(rejected — Supabase direct host is IPv6-only; the pooler is IPv4).
**Reason:** keep WDA fully isolated; the pooler is the reliable, IPv4 path.
`create_engine` sets asyncpg `statement_cache_size=0` so the transaction pooler
(PgBouncer) works. DB password + connection strings live only in fly secrets /
a local git-ignored scratch file — never committed.

---

## 2026-09-04 — Fact inserts are batched (asyncpg 32767-param limit)
**Decision:** `SqlFactRepository.add` inserts in chunks of 2000 rows.
**Reason:** A company's companyfacts can be thousands of facts; a single
multi-row INSERT (13 params/row) exceeds asyncpg's 32767 bind-parameter cap. The
live Apple demo (5923 facts) surfaced this. Batches of 2000 stay well under the cap.

## 2026-09-04 — Fact insert counts via RETURNING, not rowcount
**Decision:** `SqlFactRepository.add` counts inserted rows with
`INSERT ... ON CONFLICT DO NOTHING RETURNING id` and `len(...)`.
**Alternatives:** `CursorResult.rowcount`.
**Reason:** `rowcount` is unreliable for INSERTs into a **partitioned** table
(tuple routing reports 0), which made facts-sync report 0 inserted even when rows
landed. RETURNING counts exactly the rows actually inserted (conflicts skipped).

## 2026-09-04 — Local dev DB: native Postgres 16 + pgvector (Docker unavailable)
**Decision:** Run the dev/test database as a **native** Homebrew
`postgresql@16` service with pgvector, not the docker-compose container.
**Context:** Docker Desktop's Linux VM could not reach the registry (even
`hello-world` hung) while the host network was fine — a VM-networking stall.
**Setup performed:**
- `brew install postgresql@16 pgvector`; the pgvector bottle targets
  postgresql@**17**, so pgvector 0.8.6 was **built from source** against @16
  (`make PG_CONFIG=.../postgresql@16/bin/pg_config && make install`).
- `brew services start postgresql@16`; role `wda` (local-only credentials matching
  the docker `POSTGRES_USER`; **SUPERUSER**, needed for `CREATE EXTENSION vector`);
  databases `wda` (dev) and `wda_test` (integration tests — kept separate so
  up/down migrations never touch dev data).
- Local `.env` (git-ignored) points `WDA_DATABASE_URL` at `.../wda`.
**Reason:** Host network works; this unblocks Phase 1 immediately and gives a
real DB to explore. docker-compose (pg16 + pgvector) remains the documented,
canonical path — switch back once Docker's VM networking is healthy.
**Alternatives:** wait for Docker; use postgresql@17 (pgvector bottle matches it,
but deviates from the pg16 target).

## 2026-09-04 — Integration tests run Alembic in a subprocess
**Decision:** `tests/integration/test_migrations.py` shells out to
`python -m alembic` rather than calling `command.upgrade` in-process.
**Alternatives:** in-process Alembic API.
**Reason:** `env.py` uses `asyncio.run()`; in-process, asyncpg connections were
GC'd across event loops and raised `ResourceWarning`, which our strict
`filterwarnings=error` turned fatal (and escaped module-level filters via
pytest's unraisable hook). A subprocess isolates Alembic's loop and mirrors
`make migrate`.

## 2026-09-04 — Phase 0 exploration findings (answers to edgar-ingestion.md §5)
Run of `scripts/explore_edgar.py` against AAPL (320193), MSFT (789019),
NVDA (1045810), Build-A-Bear (1113809, small-cap) and Reddit (1713445, 2024 IPO).

1. **Filing counts / pagination.** Established filers index thousands of
   submissions (AAPL 2246, MSFT 4492, NVDA 2476, BBW 1522); `filings.recent`
   caps at ~1000 and older filings live in `filings.files[]` pages that **must
   be merged**. Reddit (recent IPO) indexes 478 in one page; its first 10-K
   (FY2024) was filed in 2025. → `SubmissionsSyncService` must follow `files[]`.
2. **10-K primary document.** A single inline-XBRL `.htm` named
   `<ticker>-<period>.htm` (e.g. `aapl-20240928.htm`, ~1.4 MB, `isInlineXBRL=true`).
   The `size` in submissions (~9.7 MB) is the **full submission** `.txt`, not the
   primary doc. → store the primary `.htm`; don't confuse it with the full-submission size.
3. **fp/fy + concepts + restatements.** `fy`/`fp` are populated and consistent
   (FY, Q1–Q3). Restatements are **pervasive**: NVDA `Revenues` has 280 facts with
   ~65 period-ends re-reported under ≥2 accessions (comparatives). Confirms the
   append-only fact design (US-2.1). Revenue concept varies by era — older filers
   use `us-gaap:SalesRevenueNet`, modern use
   `RevenueFromContractWithCustomerExcludingAssessedTax`; plain `Revenues` is
   sparse → canonical revenue needs a **fallback chain** (US-2.2).
4. **Daily index mapping.** A `master.idx` row is `CIK|Company|Form|Date|<accn>.txt`
   — the accession derives cleanly from the filename, but the row carries **no
   primary-document name**. `primaryDocument` is authoritative from *submissions*,
   not `index.json` (which lists every file incl. exhibits and `R*.htm` renders
   without flagging the primary — the explorer's first-`.htm` heuristic grabbed an
   exhibit until fixed). → `FilingFetchService` resolves the primary doc from the
   submissions metadata it already stored, never by guessing from `index.json`.

## 2026-09-04 — `edgartools`: build, don't borrow
**Decision:** Do not adopt `edgartools`; keep the small, fully-tested pure parsers
over raw SEC JSON.
**Alternatives:** depend on `edgartools` for statement assembly.
**Reason:** Our parsers already yield the exact point-in-time tuple we need
(`end`/`filed`/`accn`) with zero extra dependency. `edgartools` is heavy, imposes
its own models, and would blur the raw↔derived layer boundary. Revisit only if
canonical statement assembly (US-2.2) proves hard by hand.

## 2026-09-04 — Request logs go to stderr
**Decision:** `configure_logging` writes structlog output to `stderr`.
**Alternatives:** stdout (structlog's default).
**Reason:** The explorer emits data (incl. `--json`) on stdout; logs on stdout
corrupt it. stderr keeps stdout clean for both the CLI and piped server use.

## 2026-09-04 — EDGAR fixtures are trimmed real payloads, captured directly
**Decision:** Capture live submissions/companyfacts/index/idx for AAPL/MSFT/NVDA
with a throwaway script that dogfoods `RateLimitedClient`, then trim to a few KB
each (curated filings; ~4 concepts incl. a restatement) and commit under
`tests/fixtures/edgar/`.
**Alternatives:** commit full raw payloads (companyfacts are 3–5 MB — bloats the
repo and trips the large-file pre-commit hook); hand-author synthetic fixtures
(risks diverging from real SEC shapes).
**Reason:** Real shapes, tiny footprint, offline unit tests. The explorer's
`--save-fixtures` (story 0.4) will regenerate/extend these later.

## 2026-09-04 — Fact values stored as Decimal via `Decimal(str(v))`
**Decision:** Parse XBRL `val` through `str()` into `Decimal`.
**Alternatives:** `float`; `Decimal(float)`.
**Reason:** CLAUDE.md forbids float for facts; `Decimal(str(2.18))` preserves the
reported value exactly, `Decimal(2.18)` does not.

## 2026-09-04 — `edgartools` build-vs-borrow: deferred to story 0.4
**Decision:** Built raw parsers from scratch for 0.3; the formal `edgartools`
evaluation belongs to the explorer story (0.4) where statement assembly can be
compared on the five companies. Recorded here so it isn't lost.

## 2026-09-03 — `uv` installed via Homebrew on the dev machine
**Decision:** Use `uv` (0.12.x) as the single env/dependency manager, installed
with `brew install uv`.
**Alternatives:** pip + venv, Poetry, PDM.
**Reason:** Mandated by CLAUDE.md ("uv for env/deps"). Fast, lockfile-based,
one tool for venv + deps + running.

## 2026-09-03 — `uv.lock` is committed
**Decision:** Commit the lockfile.
**Alternatives:** gitignore it (library convention).
**Reason:** This is a deployed application; reproducible installs across dev,
CI, and the fly.io image matter more than downstream flexibility.

## 2026-09-03 — `make dev` uses uvicorn's `--factory`
**Decision:** Run the API as `uvicorn wda.app:create_app --factory` rather than
exposing a module-level `app` object (`wda.app:app`).
**Alternatives:** Module-level `app = create_app()`.
**Reason:** A module-level `app` would call `get_settings()` at import time,
requiring `WDA_*` env vars just to *import* `wda.app` — which would break unit
tests that import the module. The factory keeps import side-effect-free; tests
inject a `Settings` instance directly. (Minor deviation from the command shown
in CLAUDE.md; behaviour is identical.)

## 2026-09-03 — pytest excludes `integration` by default
**Decision:** `make test` runs `pytest -m "not integration"`; `make test-int`
runs `-m integration`. `filterwarnings = ["error"]` makes warnings fatal.
**Alternatives:** Separate test directories only; warnings as warnings.
**Reason:** Enforces CLAUDE.md rule 2 (unit tests never touch network/DB) at the
runner level and surfaces deprecations early.

## 2026-09-10 — MCP endpoint speaks OAuth 2.1 (US-10.3)
**Decision:** Turn the `/mcp` endpoint into an OAuth 2.1 authorization server
(PKCE, `client_secret_post`) using the `mcp` SDK's built-in AS. We implement only
the `OAuthAuthorizationServerProvider` (`wda/mcp/oauth.py`): **one fixed
confidential client** (id + secret via `fly secrets`) and **stateless self-encoded
JWT tokens** (HS256) for authorization codes / access / refresh. Dynamic client
registration is disabled. The legacy static `WDA_MCP_TOKEN` is still accepted as an
access token. Enabled only when `WDA_OAUTH_{ISSUER,CLIENT_ID,CLIENT_SECRET,SIGNING_KEY}`
are all set; otherwise the endpoint stays in bearer-token mode.
**Alternatives:** hand-roll `/authorize` + `/token` (crypto risk, reinvents the
SDK); DCR with a Postgres client store (needed for zero-config auto-registration —
deferred); opaque tokens in Postgres (needs shared storage — defeats stateless
multi-machine).
**Reason:** Grok and claude.ai mobile connectors only speak OAuth and reject a
static bearer token. JWT tokens verify on any fly machine with no shared store (the
same reason the transport is `stateless_http=True`); possession of the client
secret is the real access gate, keeping the endpoint as private as the bearer
token. To make the RFC 9728 well-known paths line up (`/.well-known/…` and
`/authorize` `/token` at the site root, matching `build_resource_metadata_url` and
Grok's endpoint fields), the SDK app is mounted at `/` with the transport at `/mcp`
and `/health` registered first. Signing keys < 32 bytes are rejected at startup
(HS256/RFC 7518 §3.2). Trade-off: JWTs can't be revoked before expiry; TTLs are
short (code 60s, access 1h, refresh 30d).

## 2026-09-25 — Macro sources: FRED as Treasury's distribution, and what a vintage means

**Decision.** Rates, the yield curve, mortgage rates, credit spreads, CPI and
unemployment come from FRED; crude and natural gas from EIA. No direct Treasury adapter.

**Why not Treasury.** Treasury *originates* the par yield curve, but FRED *distributes*
it, carrying every tenor from one month to thirty years, daily, current to the prior
session. A second adapter would be a second failure mode for identical numbers and would
have to re-solve point-in-time handling that already works. Verified against the live API
before deciding, not assumed.

**Why this does not breach the primary-sources rule** that keeps news and analyst
estimates out: these are statistical agencies publishing their own measurements — the
same category as a company publishing its own filing. What stays excluded is anybody's
*opinion* about what the numbers mean. One series carries a caveat: `BAMLH0A0HYM2` comes
from ICE BofA, an index provider rather than an agency. It measures observable bond
prices, so it sits inside the rule, but it is the only series here that is not a
government statistic.

**Which series, and why not more.** Four tenors (3mo/2y/10y/30y) describe the curve's
shape; 1mo, 6mo, 1y, 7y and 20y add granularity no fundamental thesis reaches for. The
30-year mortgage earns its place on constituency — the universe holds 125 REITs, 17
homebuilders and 6 mortgage lenders whose revenue is a direct function of it. The same
guardrail `technical-analysis.md` §5 applies to indicators applies here: a curated
handful beats a zoo, and every series has to be asked for by a question somebody poses.

**Two limits that are permanent until someone does more work, and are stated in the
server instructions rather than left to be discovered:**

- **Vintages are null.** Neither agency gives a per-observation release date on the
  routes in use. FRED's default endpoint stamps every row with the date of the *request*
  — a decade of daily yields all carrying that morning's date — which is why reading it
  as a publication date silently emptied every historical `as_of` query until it was
  caught. EIA publishes none at all. Real vintages need ALFRED-style requests returning
  every revision of every point. So `as_of` excludes readings dated after it but cannot
  exclude a figure revised since.
- **`BAMLH0A0HYM2` starts in 2023**, not ten years back: FRED licenses it on a rolling
  window. A fine "what is risk appetite now" gauge and a poor "what did this look like in
  a downturn" one, because the sample contains no recession.

**Not built.** A macro *check* type. A series is something you look up, not something
that watches itself — "the 10-year stays below 5%" cannot yet be a thesis condition.
