# Commander — Solution Coverage Map

*A master list of everything a complete solution for Commander needs to address, across every
layer (scheduling, message production pipeline, data retrieval, and the application as a
whole), organized by concern area rather than by component. The point of this document is to
see the whole shape before deciding how to split it across files (one document vs. an umbrella
plus per-component documents — that decision is deliberately deferred).*

*Status markers: ✅ covered in an existing solution document · 🟡 partially covered / assumed but
not stated explicitly · ❌ not covered anywhere yet · — not yet assessed (data-retrieval isn't
drafted).*

---

## I. Per-component functional scope

What each layer is responsible for, its boundaries, and how it hands off to the next.

- ✅ Scheduling: cadences, window computation (rolling/boundary/calendar-day), daylight saving,
  misfire policy, trigger model, admin control surface (Run now/Backfill/Pause/Resume/Status).
- ✅ Message production pipeline: the three trigger paths (scheduled/on-demand/PHT), the
  to-do-list + outbox pattern, the bundling rule, duplicate prevention via fingerprint, recovery
  via the watchdog.
- — Data retrieval: resolving a configuration's agreement scope (payment types, accounts,
  aliases) from Scheduling/the pipeline's request. Not drafted yet.
- 🟡 Recipient identity resolution (how a configuration resolves to the actual recipient
  type/value/name in the message) — not yet assigned to any document; most likely belongs in
  data retrieval.
- 🟡 The on-demand request contract itself (queue-based, list of config ids) is defined in the
  pipeline document, but the *caller's* side of that contract (who sends it, how they get config
  ids, any validation before it's accepted) isn't described anywhere.

---

## II. Reliability & resilience

- ✅ Pod crash recovery (scheduling: Quartz clustering + `requestRecovery`; pipeline: the
  watchdog for scheduled runs, queue redelivery for on-demand/PHT).
- ✅ Duplicate prevention (fingerprint / `UQ_Outbox_Identity`, `UQ_Run_ScheduledSlot`).
- ✅ Sustained DB/MQ outage vs. one-off transient failure, decision against an automatic
  circuit-breaker (see `db-mq-failure-analysis.md`).
- ❌ JMS acknowledgment mode — the redelivery-based safety story for on-demand/PHT recovery is
  only true under manual/client acknowledgment; not yet an explicit decision (see
  `feedback.md` #1).
- ❌ Graceful shutdown vs. crash — every pod exit is currently treated as an unplanned crash;
  no decision on whether a clean drain on `SIGTERM` is worth adding for routine rolling deploys
  (see `feedback.md` #6).
- ❌ Health checks — readiness/liveness probe behaviour isn't specified anywhere (e.g., should
  readiness fail if the DB is unreachable, so the platform stops routing to that pod?).
- ❌ Disaster recovery / business continuity beyond a single outage (a full datacenter or region
  event) — not addressed; worth at least an explicit statement of what's out of scope.

---

## III. Observability & monitoring

- ❌ Structured logging convention — not addressed anywhere.
- ❌ A correlation/run identifier propagated end-to-end through logs and the message itself
  (see `feedback.md` #2) — previously flagged externally at 6.5/10 for exactly this gap.
- ❌ Metrics (throughput, queue depth, poison rate, recovery frequency) — not addressed.
- ❌ Alerting strategy beyond the individual alert-worthy events already named in the component
  documents (`FAILED_POISON`, `ABANDONED`, the new infrastructure-failure aggregate alert) —
  no discussion of where alerts go, who owns them, or dashboards.
- 🟡 The infrastructure-vs-data failure classification (decision 11 in the pipeline document)
  is a building block for this, but the observability story around it isn't built out.

---

## IV. Security

- ❌ Secrets / credential management for DB and MQ — not addressed anywhere (see `feedback.md`
  #4; flagged previously as the single critical finding in the legacy codebase).
- 🟡 Admin endpoints are described as "authenticated, admin-only" in both documents, but *how*
  — what authentication/authorization mechanism, who is allowed to call them, whether actions
  are audit-logged — is not specified.
- ❌ Data protection for the banking data in transit/at rest (account numbers, aliases,
  balances) — not discussed.
- ❌ Audit trail for admin actions (who paused a schedule, who triggered a backfill, who
  redrove a poison item) — not discussed; related to the observability gap above but distinct
  (this is about accountability, not debugging).

---

## V. Data management

- ❌ Retention / archival policy for `Run`, `WorkItem`, `Outbox` — the outbox in particular is
  currently described as growing forever (see `feedback.md` #3).
- 🟡 `ProcessedInboundMessage` retention is at least flagged as an open question in the pipeline
  document, tied to the queues' actual redelivery/backout configuration.
- ❌ Schema / migration ownership — who owns changes to the four tables, how migrations are
  versioned and rolled out, isn't addressed.
- ❌ Compliance / data residency / audit requirements specific to financial reporting data —
  not discussed at all; worth at least an explicit scoping statement (applicable or not).

---

## VI. Performance & scalability

- 🟡 Some sizing appears implicitly (paging at ~500 configs at a time, specific frequency
  cadences), but there's no stated ceiling on message size, bundle size, or overall throughput
  target (see `feedback.md` #7).
- ❌ IBM MQ's message-size limit (100MB, per the earlier requirements review) is never connected
  to the bundling rule — an unusually large bundled configuration has no documented failure mode.
- ❌ Load/capacity assumptions (how many pods, expected peak config count, expected message
  volume) — not stated as an explicit NFR anywhere.

---

## VII. Deployment & operations

- 🟡 Multi-pod, shared-DB, shared-queue deployment is assumed throughout, but deployment
  topology itself (how many pods, environment differences dev/test/prod, config management
  across environments) isn't described.
- ❌ Rolling-deploy behaviour (see II, and `feedback.md` #6).
- ❌ Environment/configuration management beyond the TEST-only `cron-override` already
  documented for scheduling.
- 🟡 Operator runbooks exist informally (Pause during a DB issue, backfill afterward — see
  `db-mq-failure-analysis.md`), but aren't collected anywhere as a formal runbook a person could
  follow during an incident.
- ❌ Pipeline-side admin surface — redriving a `FAILED_POISON` item, inspecting/cancelling a
  stuck on-demand run, managing the dead-letter queue (see `feedback.md` #8). Scheduling has a
  rich admin story; the pipeline has none of its own yet.

---

## VIII. Integration contracts

- 🟡 The message contract (`ReportMessage`) has some fields named (in the technical companion
  material), but no documented schema-evolution/versioning strategy (see `feedback.md` #5).
- 🟡 The Executor contract (dedup on fingerprint, accepts semantically-equal messages, no
  ordering dependency) is stated as a requirement *on* Executor, explicitly not yet confirmed
  with the team that owns it.
- ✅ The scheduling → pipeline handoff (the five facts) is fully specified.
- — The pipeline → data-retrieval contract (what's requested, what's returned) isn't specified
  yet, pending that document.

---

## IX. Testing strategy

- ✅ Fast-cadence testing for scheduling (dense test schedules, still producing fresh messages
  via the `execution_id` derivation) is well covered.
- ❌ Integration/contract testing with Executor — not discussed beyond the open item to confirm
  assumptions with that team.
- ❌ Testing strategy for the failure scenarios just designed (DB outage, MQ outage, degraded
  DB) — no mention of how these would actually be validated (chaos testing, fault injection,
  etc.) before going live.

---

## X. Documentation & operational runbooks

- ❌ No formal, consolidated operator runbook exists yet — the pieces are scattered across
  `db-mq-failure-analysis.md` and the reliability sections of each component document.
- ❌ No glossary/shared-terminology document across all three components (though this session's
  own terminology-consistency passes are a step in that direction).

---

## Legend recap

✅ = covered · 🟡 = partially covered or implicit · ❌ = not covered · — = not yet assessed
(data-retrieval not drafted)

**Count:** of the 42 items above, 7 are fully covered (✅), 10 are partial or implicit (🟡), 23
are genuinely open (❌), and 2 can't be assessed yet since data-retrieval isn't drafted (—).
That's the real shape of what's left — useful to see all at once before deciding whether this
becomes one document, an umbrella plus three, or something else.
