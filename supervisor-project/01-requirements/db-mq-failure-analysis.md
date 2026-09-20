# DB and MQ Failure Handling — Analysis

*Why this exists: both existing solution documents (scheduling, message production pipeline —
and, by extension, the not-yet-drafted data-retrieval one) heavily document Kubernetes-style pod
death and recovery, but never distinguished a **sustained infrastructure outage** (SQL Server
unreachable, IBM MQ broker unreachable) from a **one-off transient failure** (a single query
timeout, a single publish retry). This was confirmed, by reviewing every design and requirements
document, to be a genuine gap that was never noticed — not a deliberate scope decision.
`requirements-gaps.md`, which exists specifically to catalogue every omission in
`requirements_v01.md`, never lists it either.*

---

## The three scenarios

**a. Total SQL Server outage.** Quartz's `JDBCJobStore` lives in SQL Server. If SQL Server is
unreachable, Quartz cannot determine what's due, cannot coordinate cluster locking, and cannot
fire anything — the whole application is effectively down. There is no application-level
mechanism that resiliences around a fully unreachable database; this is not a gap to close, it's
an inherent and correct consequence. The one thing worth stating explicitly: this needs its own
infrastructure-level alerting, because Commander's own alerting (which itself depends on
Commander being able to run) won't fire when Commander itself can't run.

**b. A known, detected DB issue (incomplete data, an ongoing outage).** The intended response is
operational, not automatic: an operator pauses the affected schedules using the already-designed
Pause admin action, waits for the database to be confirmed healthy, then resumes using the
matching Resume action. This scenario doesn't need new machinery — it needs the *existing*
Pause/Resume mechanism explicitly connected to "we know the DB has a problem" as one of its
named use cases, rather than left implicit under a vaguer "environment issue."

**c. Temporary issues (a dropped connection, a one-off DB error).** These should be — and
already are — handled automatically by the application itself. The message production
pipeline's retry/attempt-count/`FAILED_POISON` path, and data-retrieval's tri-state resolution
(`found` / `confirmed-absent` / `query-failed`), already exist precisely for this. No new
mechanism is needed; this just needs to be stated explicitly as the intended coverage, rather
than left implicit.

---

## Two risks in the gap between (a) and (b)

Neither "totally down" nor "one-off blip" quite covers the middle case: the database is up but
**degraded** — slow, connection pool under pressure, intermittently timing out. Under the
current design, every one of those failures just walks the normal per-item retry path toward
`FAILED_POISON`. Two consequences are worth naming explicitly:

1. **A degraded DB can look like "200 unrelated broken items" rather than "one systemic
   problem."** Because each failing `WorkItem` alerts individually as `FAILED_POISON`, an
   operator watching the alert stream may not realize quickly that they should invoke the pause
   runbook from scenario (b) at all. The signal that would tell them "this is systemic, go
   pause" isn't currently distinguished from "this item's data is just bad."

2. **Retrying into a degraded database can make it worse.** A straightforward retry storm piles
   more load onto an already-struggling database right when it can least handle it.

---

## Recommendation: no circuit-breaker

A full automatic circuit-breaker (detect degradation, auto-pause processing, auto-resume on
recovery) was considered and deliberately **not** recommended. It's real complexity — a state
machine, thresholds, coordinating "breaker open" state across every pod in the cluster — for a
benefit that is mostly *faster reaction time* during what should be a rare event. The existing
safety nets (the durable outbox, the do-nothing misfire policy, manual Pause/Resume, and poison
alerting) already prevent data loss or duplication without one.

Instead, six lightweight additions — all either making an already-structurally-possible thing
explicit, or a small, uncontroversial recommendation:

1. **Classify failures as infrastructure vs. data at the point they're logged and alerted, and
   raise a distinct, aggregate alert for a spike of infrastructure-classified failures** —
   separate from individual `FAILED_POISON` alerts. Data-retrieval's tri-state already gives
   this distinction for free (`query-failed` is already a separate outcome from
   `confirmed-absent`); it just needs to be explicitly used to drive a systemic-looking signal,
   so an operator recognizes a DB problem fast enough to invoke scenario (b), instead of seeing
   what looks like many unrelated broken items (risk 1, above).
2. **Explicitly document the total-DB-outage behaviour for scheduling** (scenario a) and connect
   it to the existing do-nothing misfire policy — a DB outage is the single most likely
   real-world cause of "the whole cluster is down," and the existing recovery/backfill story
   already covers it; it just needs to be said explicitly rather than left as a generic
   "cluster down" case.
3. **Explicitly document what happens when the relay can't reach MQ itself at publish time** —
   this is already handled gracefully by the durable outbox design: the row simply stays
   `PENDING` and the relay's normal loop retries it once MQ is reachable again. No new mechanism
   needed, just a sentence confirming the behaviour, since nothing currently states it.
4. **A reassurance note on heartbeats.** A pod's heartbeat write failing during a DB outage is
   safe, even if the watchdog later perceives several runs as "stale" all at once once the
   database returns — the CAS-based single-claim on `heartbeat_at` still ensures only one pod
   ever actually resumes a given run.
5. **Recommend backoff-with-jitter** (not fixed-interval) for DB and MQ retry logic, so recovery
   from an outage doesn't itself create a synchronized "thundering herd" against the
   just-recovered dependency (risk 2, above).
6. **Connect Pause/Resume's documented use case explicitly to "operator has been informed of a
   DB or MQ issue"** (scenario b), not just the vaguer "environment issue" language currently
   used.

---

## Open item — not yet resolved

**Pause/Resume only covers scheduled firings.** Pause and Resume are scheduling concepts, keyed
on `(report_type, frequency)` — they only stop **scheduled** work. On-demand requests and PHT
pushes arrive via MQ, not Quartz, and are entirely untouched by Pause. During a degraded-DB
incident, pausing scheduled work does not stop on-demand/PHT traffic from continuing to hit the
same degraded database — those requests would keep failing and rely on the queue's own
redelivery as a safety net. This is self-limiting and safe (no data loss, no duplication), but
potentially noisy, and it means an operator's Pause action during a DB incident is only a partial
mitigation. Worth a decision from the team: is this acceptable as-is, or should there be a way to
pause on-demand/PHT intake too during a known incident?

---

## Where this lands in the solution documents

**Status 2026-09-21: fully covered**, after the original `01-commander-scheduling.txt` /
`02-commander-message-production-pipeline.txt` were superseded by the current documents, and a
gap this introduced was caught and closed.

- `commander-scheduling.md`: scenario (a) and the Quartz/SQL Server dependency are an explicit
  bullet in Section 6 (Assumptions) — this was dropped when the document was rebuilt fresh from
  the original design source material (which predates this analysis), then re-added. Recommendation
  6 (Pause/Resume tied to "informed of a DB or MQ issue") is in Section 5, B (Admin actions).
- `solution-document-commander.md` (Assembly/Delivery): scenario (c) is implicit in the
  WorkItem retry/Failed-poison path (1.7, C). Recommendation 1 (infra-vs-data classification +
  aggregate alert) is 1.4, decision 13. Recommendation 3 (outbox-stays-pending when MQ is
  unreachable) is in 1.6/1.7, Delivery. Recommendation 4 (heartbeat reassurance) and the open
  item (Pause/Resume's scheduled-only scope) are bullets in 1.7, Assembly E. Recommendation 5
  (backoff-with-jitter) covers both MQ retries (1.6/1.7, Delivery) and DB retries (1.4, decision
  14, added specifically to close a gap where only the MQ side had been stated).
- `commander-data-retrieval.md`: exists now. Recommendation 1's structural hook (the tri-state —
  found / confirmed-absent / query-failed) is documented in Section 2 and Section 7.6.
