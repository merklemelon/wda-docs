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

## Why the build command looks like that

Vercel's build image ships a **uv-managed** Python, which refuses `pip install` into the
system environment under [PEP 668](https://peps.python.org/pep-0668/):

```
error: externally-managed-environment
╰─> This Python installation is managed by uv and should not be modified.
```

So the build asks `uv` first and falls back to `pip --break-system-packages` only if uv
is absent — rather than hard-coding either and depending on an image detail that is not
ours to control.

It ends with `ls -la _site/index.html` on purpose. A build that produces an empty
directory and reports success publishes a site that 404s on every path, which is exactly
what happened here before the repository was connected correctly. Failing loudly is
better than deploying nothing quietly.
