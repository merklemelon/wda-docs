# WorldDominationApp

An equity-research backend. It ingests SEC EDGAR filings and market data for the US
public-company universe, normalizes them, enriches them with LLM-extracted narrative
signal, and serves the result to a language model over MCP.

These are its working documents: the design notes written while building it, and the
field guide written for using it.

## The one idea

**Everything traces to a primary source.** Every number comes from a document a company
filed, a price, or a statistical agency's own measurement. There is no news feed, no
analyst estimates, no ratings.

That is a deliberate boundary, not a gap. It means an answer can be *audited* rather than
trusted — every row carries the accession number, the form type and the filing date. It
also means the system is weak wherever the answer lives in the market's opinion of a
filing rather than in the filing, and the notes here say so where that bites.

## Where to start

| If you want | Read |
|---|---|
| What it does, in one pass | [Field guide](field-guide.html) |
| How it is built | [Overview](architecture.md) |
| How a case becomes checkable | [Thesis layer](thesis.md) |
| What the price already assumes | [Forward analysis](forward-analysis.md) |
| Why something was decided the way it was | [Decisions](decisions.md) |

## A note on what is here

The design notes are written for whoever is changing the code, the field guide for
whoever is using the system. They are candid about limits, wrong turns and things that
are not built — a note that oversells is worse than no note, because the next person
trusts it.

The application source is private; this repository is documentation only.
