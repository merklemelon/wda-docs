# WorldDominationApp — documentation

The design notes and field guide for WDA, an equity-research backend that ingests SEC
EDGAR filings and market data for the US public-company universe, normalizes them,
enriches them with LLM-extracted narrative signal, and serves the result to an LLM over
MCP.

Published at **https://wda-docs.vercel.app** — built by Vercel from this repo on push.

## What is here, and what is not

This repo is **public and documentation only**. The application source is private.

Also absent by choice: the working progress log, the development plans, acceptance
criteria, and the superseded v0.4 vision document. Those are internal notes about what
broke and when — useful to whoever maintains the system, noise to a reader who wants to
know how it works.

## Building locally

```sh
pip install -r requirements-docs.txt
mkdocs serve
```

## Where these come from

The notes are mirrored from the application repo, which is where they are written and
where they change alongside the code they describe. Edit them there, not here.
