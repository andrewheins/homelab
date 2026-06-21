# AGENTS.md — [project name]

*Copy this file into any new project repo and fill in the bracketed sections.
Do not restate or override anything from the global docs below — if something
here contradicts them, the global docs win and this file is wrong.*

---

## Read first

Before touching this project, read, in order:

1. `infrastructure-contract.md` — the network-wide integration rules
2. `session-behavior.md` — what to read before acting and what checklist to
   produce before ending a session
3. `config.md` (private repo) — current real state: hosts, ports, database
   names, credential references

This file adds only what's specific to **[project name]**. Everything else —
how to start a session, how to end one, how services integrate — is already
defined in the three documents above and isn't repeated here.

---

## What this project is

[One or two sentences. What it does, not how. "Self-hosted financial
intelligence system for the household" — not the tech stack, not the
history of how it got built.]

---

## Plane and classification

- **Plane:** [monitoring / application] — per `infrastructure-contract.md` §1
- **Project or service:** [project / service] — per `infrastructure-contract.md`
  §5 Rule 5. Project if it has custom logic or might move hosts; service if
  it's third-party software run as-is.

---

## config.md sections this project touches

[List only the sections this project reads from or writes to. This is the
one place a session should look to know which parts of config.md are
relevant before editing it — not a copy of the values themselves, just
which tables.]

- `## Postgres databases` — row: `[database name]`
- [add others as needed; delete this section entirely if the project is
  stateless and touches no config.md tables]

---

## Project-specific context

[Anything a session needs to know that isn't covered by the three global
documents and isn't a config.md value. Examples: a non-obvious dependency
between this project and another; a deliberate design choice that would
otherwise look like a mistake; a known limitation. Keep this short — if it's
explaining "why" at paragraph length, it probably belongs in the project's
own design notes instead, not here.]

---

## What this file is not

Not a restatement of `infrastructure-contract.md` or `session-behavior.md`.
Not a runbook for operating this project day-to-day. Not a place for
narrative, migration history, or dates — see `session-behavior.md` §3 for
why that discipline applies to `config.md`, and apply the same standard here.