# AGENTS.md — this repo

*Read this before adding, editing, or removing any file here. This repo is
documentation, not code — the rules below exist because documentation drifts
silently in a way code doesn't: nothing fails a build when a doc goes stale or
leaks the wrong detail.*

---

## 1. What this repo is

A small set of self-contained architecture documents describing how this
network and its agentic systems are designed to work — topology, integration
rules, and reusable patterns (the ask-queue model, Honk, the two-plane split).

It's public on GitHub deliberately. The ideas here are meant to be legible to
someone with no other context, and visible enough to invite outside scrutiny.
That's a design goal, not an accident — treat every document as something a
stranger might read with zero prior context.

---

## 2. What belongs here

- Architecture: topology, planes, integration rules, reusable design patterns
- Anything written as **criteria**, not as a snapshot of current hardware —
  "the monitoring plane must survive application-plane failure," not "the Pi
  must survive Omar going down"
- Documents that stay true even after the hardware underneath them changes

## 3. What does not belong here

- Real hostnames, IP addresses, internal DNS names
- Credentials, tokens, check UUIDs, topic names — anything that functions as
  a secret
- Specifics that go stale fast: actual ports, actual schedules, actual file
  paths on a real machine
- Anything that reveals more than necessary about what's actually running —
  examples should be generic and portable, not lifted from the real stack
- Operational runbooks, backup procedures, deployment steps — those belong
  with the thing they operate, not here

The test for any sentence before it goes in: **could this help someone reach
or damage the actual systems, or will it embarrass-by-specificity in six
months?** If yes to either, it doesn't belong here — it belongs in `config.md`
(private repo) or in the relevant project's own `AGENTS.md`.

---

## 4. How to write here

- **Self-contained.** Each document should make sense to someone with no
  access to the codebase or the conversation that produced it.
- **Durable over transitional.** State the principle, not the current
  instance of it. If a sentence will be wrong the day a machine gets
  replaced, rewrite it as criteria instead.
- **State the rule, not the backstory.** A rule earns its place by being
  generally true. The incident that prompted it belongs in conversation or
  commit history, never in the rule's own text.
- **One "why" line per rule is fine, a paragraph of justification is not.**
  Existing documents (see `infrastructure-contract.md`) use a short "Why:"
  line under most rules — match that, don't expand it.
- **Generic examples only.** Illustrate with invented names, not the actual
  services running on this network.
- **Say what the document is not.** Every document should end with an
  explicit out-of-scope section — it's cheap to write and prevents scope
  creep into adjacent concerns (security policy, backup policy, deployment
  guides).

---

## 5. Relationship to the private config repo

This repo and the private `config.md` repo are deliberately separate, not
nested — see `infrastructure-contract.md` for why. As a quick check: if a
sentence here would become wrong or sensitive the moment you wrote down a
real value, that value belongs in `config.md`, referenced here only by
placeholder (`${key}`) or by description, never spelled out.

---

## 6. Update discipline

When the topology or a rule changes, **update the document first, then
change the actual systems to match it** — not the other way around. The
contract is the intended state; reality should be chasing it, not the
reverse.

`config.md` is the deliberate exception — see `session-behavior.md` §3 for
why it runs in the opposite direction.

---

## 7. Session behavior

How an agent session should start and end — what to read first, what
checklist to produce after, and why `config.md` updates stay human-applied
rather than automated — is its own document: `session-behavior.md`. It's
cross-cutting (applies regardless of which project a session touches) and
referenced from each project's own `AGENTS.md`, not duplicated into any of
them.

---

## 8. What this file is not

Not a style guide for code. Not a contributor onboarding doc. Not a
description of what any individual document currently says — read the
documents themselves for that. This file only governs what's allowed to go
into this repo and how it should read once it's there. Session behavior
lives in `session-behavior.md` (§7), not here.