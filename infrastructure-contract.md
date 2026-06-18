# Local Infrastructure Contract

*Drop this into any agent context, project, or design conversation. All services
on this network integrate according to the rules below. Update this document before
changing any integration pattern.*

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

Every service and application on this network follows all four rules.
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

---

## 7. Out of scope

This document covers integration contracts only:

- Not a security policy
- Not a backup policy
- Not a deployment guide
- Not a service-level agreement

---

*End of contract.*