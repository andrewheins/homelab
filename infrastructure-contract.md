# Local Infrastructure Contract

*Drop this into any agent context, project, or design conversation. All services
on this network integrate according to the rules below. Update this document before
changing any integration pattern. See `session-behavior.md` for how this plays out
per-session — what to read before acting and what to check before ending one.*

---

## 1. Two planes

The network runs two planes. Placement is determined by function, not by machine.

**Monitoring plane**

Watches the application plane and reports on it. Must run on hardware independent
of the application plane — its value depends on surviving failures in what it
monitors.

Criteria for monitoring plane placement:
- Must remain operational when the application plane is unavailable
- Must have a notification path that does not route through the application plane
- Low resource footprint; no compute-heavy processes

**Application plane**

Where work happens: background jobs, APIs, data stores, workflow automation, model
inference. An application plane restart pauses work — that is acceptable.

Criteria for application plane placement:
- Anything that does work rather than watching work
- All persistent data stores
- Any service with non-trivial compute or memory requirements

Distribution beyond these two planes has no architectural benefit. Fewer machines
is better. Split only when the monitoring/application boundary requires it.

---

## 2. Network monitoring

Traffic-level collection must run on the monitoring plane — it is the only node
with full network visibility.

Collection lives on the monitoring plane. Storage, querying, and audit tooling live
on the application plane. The collector ships events to the application plane; it
does not store them locally.

---

## 3. Notification transport

The monitoring plane requires a notification path that does not depend on the
application plane. If the application plane is unavailable, the monitoring plane
must still be able to deliver alerts.

ntfy runs on the monitoring plane. This gives Healthchecks a notification path
that does not route through the application plane — if the application plane is
down, the monitoring plane can still reach you.

Any service on the network may POST to ntfy directly for one-way push. Two-way
flows go through Honk (§5, Rule 2).

---

## 4. Service registry

Callable infrastructure only. Applications that consume services are not listed
here.

| Service | Plane | Role |
|---|---|---|
| Healthchecks | Monitoring | Heartbeat monitoring for all scheduled jobs and periodic tasks |
| ntfy | Monitoring | Push delivery to human; one-way only (see §3) |
| Honk | Application | Escalation queue; two-way notification and human decision routing |
| Postgres | Application | Primary persistent data store |
| n8n | Application | Workflow automation and scheduling |

Actual hostnames and addresses are declared in `config.md`, which is not committed
to version control.

---

## 5. Integration rules

Every service and application on this network follows all five rules.
Exceptions require explicit justification documented here.

### Rule 1 — All scheduled jobs register with Healthchecks

Any job, scheduled workflow, or periodic task registers a check and pings on
completion. A job that does not ping is a missed-check alert.

```bash
# Success:
curl -fsS --retry 3 https://<healthchecks-host>/ping/<check-uuid>

# Explicit failure:
curl -fsS --retry 3 https://<healthchecks-host>/ping/<check-uuid>/fail
```

Every check's Description field states what it monitors, where the job runs
from, and what to do if it fires.

Every check uses three tags:
- `plane:monitoring` or `plane:application` — which plane the job runs on.
  Combined with `config.md`'s host table, this is the access surface.
- `type:cron` or `type:heartbeat` — whether this check expects a job to run
  and ping once on completion, or a long-running process pinging on a steady
  interval to prove it's alive.
- `origin:<name>` — which project or service this check belongs to.

No exceptions. Silence is not success.

---

### Rule 2 — Escalations requiring human response go through Honk

When a process reaches a decision boundary — it has a proposed answer but needs
a human to confirm, override, or act — it publishes a honk. Direct notification
calls are not used for two-way flows.

```python
client.publish(
    honk_type="review_item",
    subject={
        "item_id": "abc123",
        "proposed": "option_a",
        "confidence": 0.71
    }
)
```

---

### Rule 3 — Fire-and-forget alerts use the notification transport directly

Informational pushes requiring no human decision may POST to ntfy directly,
without going through Honk.

```bash
curl -d "Batch complete: 83 items processed" \
  https://<ntfy-host>/ntfy/<topic>
```

If a notification later requires a response, convert it to a honk. Do not add
response handling to a direct notification call.

---

### Rule 4 — Persistent data lives in Postgres on the application plane

Data intended to persist beyond 48 hours belongs in Postgres. SQLite is acceptable
only for isolated, service-local state that no other service reads.

One database per application. No cross-application table access. Cross-service data
exchange happens via API, not shared schema.

---

### Rule 5 — Containerize once a change to its own code no longer requires a rebuild to take effect

The deciding question is mechanical, not a label: **does changing this
thing's behavior require rebuilding its container image?** If yes, native
hosting until that stops being true. If no, containerize — regardless of
how often the thing changes.

This means *rate of change* and *containerization* are independent questions.
Data riding on top of a stable engine — n8n workflow definitions, Postgres
table contents, application config — can change hourly with zero rebuild
cost, because the container never has to be rebuilt for that data to take
effect. The engine underneath stays exactly as stable as it already is. What
actually drives the rebuild cost is *source code whose own changes require a
new image* — and that's a property of what's being actively written, not of
what kind of software it is.

In practice, this sorts into two situations:

**Code you are actively writing, where a change means rebuilding the image.**
Native/systemd while this is true. Containerizing here multiplies iteration
friction by every single change — image rebuild plus container restart, in
place of editing a file and a `systemctl restart` (or, for many stacks,
nothing at all). Once the project reaches a release that holds for a real
stretch without needing a code change — not "I haven't gotten back to it,"
genuinely stable — move it to a container. The reproducibility benefit
(rebuild this exactly, on new hardware, from its definition rather than from
memory) starts paying rent exactly when you stop paying the rebuild-friction
cost on every edit.

**Everything else: third-party software you didn't write, and your own
projects once stable.** Containerize by default. This includes services
whose configuration changes constantly (n8n's workflows, a database's rows) —
none of that requires touching the image, so containerization costs nothing
here and still buys the reproducibility win.

A project, while in the actively-rebuilding-the-image phase, lives in its own
repo with an `AGENTS.md` referencing this contract (see `AGENTS.template.md`)
and an `.env.example` declaring required configuration keys — the
configuration contract, which doesn't depend on hosting mechanism and won't
go stale while the code is still moving. Real values go in a local `.env`
(gitignored, populated from `config.md`). Write the `docker-compose.yml` when
you actually containerize, against the stable thing as it really is — not
earlier, speculatively, against a target that's still moving.

Third-party software deployed as-is (Healthchecks, ntfy, n8n, Postgres, and
similar) needs only a `docker-compose.yml` and `.env` on the host that runs
it — no repo, no remote, no sync job. It has exactly one consumer, the host
it runs on; there's no second reader to distribute to.

**Exception — GPU-accelerated inference on Apple Silicon.** Container
runtimes on Apple Silicon cannot access the GPU; a containerized inference
engine silently falls back to CPU-only execution, at a severe performance
cost. Any service performing Metal-accelerated inference runs natively on
the host regardless of its development phase. Services that merely call such
an engine over HTTP (rather than performing inference themselves) carry no
such restriction and follow the rule above normally.

**Soft exception — one-shot scheduled scripts.** A short-lived, cron-triggered
script with no persistent process and nothing to isolate may remain a native
host script even once stable. Judgment call, not a rule: containerize if it
earns its keep, skip it if it's ceremony.

---

## 6. Adding a service

Before building, answer each question and update this document:

1. **Which plane?** Monitoring if it must survive application plane failures.
   Application plane otherwise.
2. **Scheduled?** Register with Healthchecks.
3. **Needs human decisions?** Use Honk. Informational only? Use notification
   transport directly.
4. **Produces persistent data?** Declare its Postgres database name here before
   writing the first table.
5. **Callable by other services?** Add it to the service registry in §4.
6. **Does changing it require rebuilding its container image?** If yes —
   you're actively writing the code — native/systemd until it stabilizes,
   with the repo/AGENTS.md/compose scaffolding from §5 Rule 5 in place ahead
   of that move. If no — third-party, or your own code once stable —
   containerize now.
7. **Apple Silicon GPU inference?** Run natively and note the exception here.

---

## 7. Out of scope

This document covers integration contracts only:

- Not a security policy
- Not a backup policy
- Not a deployment guide
- Not a service-level agreement

---

*End of contract.*