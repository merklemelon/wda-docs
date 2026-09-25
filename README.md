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

Vercel's build image ships a **uv-managed** Python. Two attempts failed before the
working one, and both failures are worth recording:

1. `pip install` is refused outright under [PEP 668](https://peps.python.org/pep-0668/) —
   *"This Python installation is managed by uv and should not be modified."*
2. `uv pip install --system` **reported success** and then `python3 -m mkdocs` could not
   import it. The package went somewhere the interpreter was not looking, and because the
   install exited zero, the fallback never ran.

So the build stopped trying to modify the environment and asked `uv` for a disposable one
instead. `uv run --with ... --no-project` resolves mkdocs into an ephemeral environment,
runs the build there, and touches nothing on the image. Nothing to install, nothing to
collide with, no dependence on where a system install lands.

It ends with `ls -la _site/index.html` on purpose. A build that produces an empty
directory and reports success publishes a site that 404s on every path — which is exactly
what happened here while the project was connected to the wrong repository. Failing
loudly beats deploying nothing quietly.
