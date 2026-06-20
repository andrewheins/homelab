# session-behavior.md

*Reference this from any project's `AGENTS.md` or from agent context directly.
Applies network-wide, regardless of which project a session is working on —
see `AGENTS.md` §2 for why a cross-cutting pattern like this belongs in this
repo rather than duplicated per project.*

---

## 1. Scope

Applies to any agent session that touches infrastructure: creates or modifies
a service, changes a connection detail, migrates data, or registers/changes a
Healthchecks check.

---

## 2. Start of session

Read `infrastructure-contract.md` (the rules) and `config.md` (current state,
private repo — see `AGENTS.md` §5) before acting. If either is unreachable,
say so before proceeding — don't fall back to memory of a prior session.

`config.md` is a snapshot, not a guarantee. When it disagrees with the live
system, the live system wins. Note the discrepancy for the end-of-session
checklist (§3) rather than trusting whichever is more convenient.

---

## 3. End of session

Produce a checklist of what changed. Do not write to `config.md` directly —
a human applies the edit.

This is asymmetric with `infrastructure-contract.md` §6 ("update the
document first, then change systems to match it") on purpose: `config.md` is
state and is expected to chase reality; the contract is intent, and reality
chases it instead. Different documents, different directions, both correct.

Checklist entries are facts, not narrative — "Postgres listens on `:5432` via
container `postgres`," not the story of how it got there — and name anything
that should NOT be written yet: in-progress, broken-on-purpose, or
deliberately deferred work. An incomplete checklist is more useful than one
that silently presents a known-broken value as current.

*Why:* config drift is invisible until something fails against it.
Automating the read side is free; automating the write side would remove the
only review step a shared, trusted file has left.

---

## 4. What this file is not

Not a runbook — it doesn't say how to operate any specific service, only how
a session should orient itself and close out. Not a replacement for a
project's own `AGENTS.md`, which still owns project-specific context (which
`config.md` sections a given project typically touches, anything beyond the
two documents in §2). Not a backup or change-management policy.