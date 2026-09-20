# Feedback — Gaps Found in a Step-Back Review

*A follow-up pass over the solution documents (scheduling, message production pipeline, and —
by extension — the not-yet-drafted data-retrieval one), looking for the same kind of gap the
DB/MQ failure-handling review found (see `db-mq-failure-analysis.md`), now with the actual
target stack in mind: Java/Spring Boot, MSSQL, IBM MQ (other frameworks/libraries allowed where
they genuinely help). Each item below was checked directly against the current text of both
existing documents before being listed — nothing here is assumed.*

*Status: to be fixed one at a time. Check an item off as it's addressed, and note which
document/section it landed in.*

---

## 1. [x] JMS acknowledgment mode — top priority, correctness-critical — RESOLVED 2026-09-20

**Resolution:** Added as `solution-document-commander.md`, 1.4 Architectural Decisions, decision
5 — on-demand and inbound-push consumers use manual (client) acknowledgment, never a framework
default, explicitly because the redelivery-based crash-recovery claim depends on it. Cross-linked
from 1.7 Assembly D, where that redelivery claim is made.

**The gap:** Both documents state that "the queue itself automatically redelivers the message
to another pod" as the safety story for on-demand and PHT recovery — no watchdog needed, MQ
redelivery covers it. That claim is only true under **manual/client acknowledgment**,
acknowledging a message only *after* its work is durably recorded. Under JMS's *default*
auto-acknowledge mode, the message is removed from the queue the moment it's delivered — before
processing even starts — so a crash mid-processing loses the message silently, with nothing left
to redeliver.

**Why it matters:** This isn't a missing nice-to-have; it's the mechanism an already-published
correctness guarantee depends on. Left implicit, a default configuration choice could silently
break a claim already made in writing.

**Affected documents:** `02-commander-message-production-pipeline.txt` (1.4 Architectural
Decisions — needs its own explicit decision; currently assumed away).

---

## 2. [x] Observability / correlation-ID propagation — RESOLVED 2026-09-20

**Resolution:** Added as `solution-document-commander.md`, 1.4 Architectural Decisions, decision
6 — a Run's identifier threads through every log line for that run, and each request's own
identity threads through the request payload itself, giving Executor the same identity to trace
by. Cross-linked from 1.6 Assembly's Outbox bullet, where that identity (the fingerprint) is
defined.

**The gap:** Zero mentions of structured logging, metrics, or a shared identifier across either
document. There's no documented way to trace one report end-to-end through scheduling → the
pipeline → Data Retrieval → the outbox → MQ → Executor.

**Why it matters:** Previously flagged by an external reviewer of the legacy codebase at 6.5/10
specifically for this reason. Painful to retrofit later, cheap to design in now — most naturally
by threading the `Run`'s `execution_id` through log lines and the message itself.

**Affected documents:** Likely all three solution documents need at least a shared convention;
the pipeline document (which already owns `execution_id`) is the natural anchor point.

---

## 3. [x] Retention / archival policy — RESOLVED 2026-09-20

**Resolution:** Added as `solution-document-commander.md`, 1.4 Architectural Decisions, decision
8. No regulatory retention floor applies (confirmed), so this is an operational default: Run
and WorkItem retained 90 days after a terminal state, except `Abandoned` Runs and Failed-poison
WorkItems retained 1 year for incident investigation; Outbox retained 1 year regardless of
outcome, as the closest thing Commander keeps to an actual delivery record. `ProcessedInboundMessage`
retention remains separately tracked as #14 (needs the real queue backout config, not a default).

**The gap:** Nothing bounds how long `Run`, `WorkItem`, or `Outbox` rows live. The pipeline
document says the outbox holds "one row per fingerprint, **ever**." At up to 10,000
configurations every 30 minutes, that's unbounded growth with no stated purge policy.
`ProcessedInboundMessage`'s retention is at least flagged as an open question; the other three
tables aren't mentioned at all.

**Why it matters:** Left undecided, this becomes a production incident (a table growing without
bound) rather than a design decision made on purpose.

**Affected documents:** `02-commander-message-production-pipeline.txt` (1.4 or 1.5; likely a new
open item pointing at Section 2 once the table schemas are documented there).

---

## 4. [ ] Secrets / credential management — user handling directly, not blocked (2026-09-20)

**The gap:** Zero mentions of how DB or MQ credentials are handled, anywhere.

**Why it matters:** Flagged in the earlier requirements review as the single *critical* security
finding in the legacy codebase (committed credentials). Even if the actual mechanism is an
infrastructure decision (e.g., a secrets manager, injected environment variables), the solution
documents should state the assumption explicitly rather than staying silent on it.

**Affected documents:** Likely a new assumption in `01-commander-scheduling.txt` and
`02-commander-message-production-pipeline.txt` 1.5 (both depend on DB; the pipeline also depends
on MQ).

---

## 5. [x] Message contract / schema-evolution strategy — RESOLVED 2026-09-20

**Resolution:** Added as `solution-document-commander.md`, 1.4 Architectural Decisions,
decisions 9 and 10. Decision 9 is the direct fix for the legacy incident: the payload is built
from its own dedicated structure, never a domain object reused elsewhere. Decision 10 sets the
ongoing evolution policy: additive-only fields, Executor ignores unrecognized ones, and a
genuinely breaking change becomes a new report type rather than an in-place restructure. See
also #16 (`ReportMessage` structure suggestion) for the proposed payload shape itself.

**The gap:** No documented approach for how the `ReportMessage` payload gets versioned safely as
fields are added later.

**Why it matters:** There's a specific historical incident behind this concern — the legacy
system broke Jackson deserialization when a shared domain class picked up new helper methods.
Worth a deliberate decision before the schema is set in stone, not an accident of whatever the
serialization library does by default.

**Affected documents:** `02-commander-message-production-pipeline.txt`, most naturally landing
once Section 2 defines the actual message schema — but the *strategy* (the decision) belongs in
Section 1.

---

## 6. [ ] Rolling-deploy vs. crash (graceful shutdown)

**The gap:** Both documents frame every pod exit as an unplanned crash ("any pod can be killed at
any moment... with no warning and no graceful shutdown"). That's the correct worst case to design
for, and it does make routine rolling deploys safe by construction — but it means every deploy
currently pays the full crash-recovery cost (watchdog timeout, MQ redelivery visibility timeout)
rather than draining cleanly on `SIGTERM`.

**Why it matters:** Not a correctness gap — a deliberate efficiency/operations call. Deploys
happen constantly; crashes should be rare. Worth an explicit decision either way rather than
defaulting into "every deploy looks like an incident."

**Affected documents:** `solution-document-commander.md` (1.4, as a new decision or an explicit
note on the existing crash-handling decisions); possibly `commander-scheduling.md` too, for
Quartz's own shutdown behaviour.

**Status 2026-09-20 — discussed, not yet drafted, revisit later:** User has already added a
Spring Boot `server.shutdown=graceful` property, and Kubernetes deployments are confirmed
rolling. Important nuance raised: that property only covers the embedded web server (stops
accepting new HTTP requests, waits for in-flight ones) — most of what actually needs draining
here isn't HTTP-triggered, so real work remains beyond that one property:

1. **Delivery's drain loop** — a continuous background loop, not HTTP-triggered. Needs its own
   shutdown hook: stop claiming new Outbox rows, finish sending whatever's already claimed, then
   exit. The graceful-shutdown property does nothing for this on its own.
2. **On-demand/PHT JMS listener containers** — their own separate shutdown lifecycle from the
   HTTP server. Need explicit configuration to stop accepting new messages but let the one
   currently being processed finish (pairs with decision 5's manual-ack requirement).
3. **Quartz's own shutdown setting** (`waitForJobsToCompleteOnShutdown`) — entirely separate
   from Spring's HTTP graceful shutdown. Without it, a scheduled Run mid-page (or the watchdog
   mid-tick) can be interrupted abruptly regardless of the web-server setting.
4. **`terminationGracePeriodSeconds` on the Kubernetes deployment** — needs to be long enough
   to cover the worst realistic case (finishing a page of ~500 configs, or an in-flight
   publish). Too short, and Kubernetes sends `SIGKILL` before any of the above finishes anyway.
   Not yet confirmed whether this is set explicitly or left at the Kubernetes default (30s).
5. **Stop claiming *new* work the moment shutdown starts, not just at the end of the grace
   period** — a pod already shutting down shouldn't pick up a new scheduled firing, on-demand
   message, or Outbox row to send, even in the seconds before it exits.

Revisit and draft into 1.4 once ready, most likely as its own decision covering all five points
(or a subset, if some are already handled by infrastructure config not reflected here).

---

## 7. [x] Sizing / throughput bounds — RESOLVED 2026-09-20 (message-size ceiling; throughput bounds not addressed)

**Resolution:** Added as `solution-document-commander.md`, 1.4 Architectural Decisions, decision
7 — IBM MQ's 100MB message-size limit applies specifically to Bundled requests (the one shape
with no upper bound on what it merges); a request that would exceed it fails immediately as
poison rather than being sent, alerting an operator, since the size is deterministic and retrying
wouldn't help. Cross-linked from the bundling rule (1.7 B) and the WorkItem Failed-poison state
(1.7 C). Note: this closes the message-size half of the gap; a throughput *target* (messages/sec,
pods needed) is still not addressed anywhere — that's closer to a capacity-planning /
performance-testing input than a design decision, and is being left open rather than guessed at.

**The gap:** No message-size ceiling, no bundle-size ceiling, no throughput target anywhere.
IBM MQ has a real message-size limit (100MB, per the earlier requirements review), but nothing
in the current documents connects the bundling rule to it — an unusually large bundled
configuration could theoretically produce a message that's too big.

**Why it matters:** Without a stated ceiling, "bundle everything into one message" (the current
bundling rule) has no documented failure mode for the case where that message would be too
large to publish.

**Affected documents:** `02-commander-message-production-pipeline.txt` (1.4/1.7.B, the bundling
rule).

---

## 8. [ ] Assembly/Delivery-side admin surface, and the retry/dead-letter jobs themselves

**The gap:** Scheduling has a rich, well-designed admin story (Run now, Backfill, Pause, Resume,
Status). Assembly and Delivery have none of their own — no documented way to redrive a
`FAILED_POISON` WorkItem, inspect or cancel a stuck on-demand run, or manage the dead-letter
queue. This is two gaps, not one: the admin *surface* (an operator's way to act on these), and
the underlying *mechanism* itself — what job actually retries a failed publish, what job
processes the dead-letter table, on what schedule, with what backoff — neither is designed yet,
even though `DeadLetterRecoveryJob` is mentioned in the broader requirements as a known concept.
Delivery's current description (`solution-document-commander.md`, 1.6/1.7) states that a
delivery failure "is handled through dead-letter recovery" without saying what that job does.

**Why it matters:** Operators will need to act on Assembly/Delivery-level problems (poison
items, a stuck run, a growing dead-letter queue) the same way they already can for schedules —
and without the underlying job(s) designed, "dead-letter recovery" is currently just a name.

**Affected documents:** `solution-document-commander.md`, Assembly and Delivery (1.4/1.6/1.7).
Explicitly deferred until Assembly is further solutioned — noted 2026-09-19, not to be drafted
prematurely.

---

## 9. [x] Recipient identity resolution — RESOLVED (placement) 2026-09-20

**Resolution:** `commander-data-retrieval.md` now exists and establishes where and how this
happens: every configuration resolves to exactly one recipient, and recipients are resolved in
the same batched way as everything else — the distinct recipient ids across a whole page are
looked up in one read, not one lookup per configuration (Section 2, "The recipient"). This
closes the original gap ("not specified where it happens").

**Still open:** the exact recipient field contract (type, value, name — the specific columns
carried into the request) is Implementation Reference-level detail, deliberately deferred along
with the rest of that document's Section 7, same phased approach used for
`commander-scheduling.md`. Revisit once that section is built.

**Affected documents:** `commander-data-retrieval.md`, Section 2 (placement) and Section 7,
not yet built (field contract).

---

## 10. [x] The scheduling/pipeline boundary is misdrawn — RESOLVED 2026-09-20

**Resolution:** Resolved by construction in `solution-document-commander.md`. Scheduling's 1.7 A
now does only two things — pick up the clustered firing, invoke Assembly's scheduled entry
point — and stops. Assembly's own 1.7 A owns the full resolve-window → create-Run → check-flag
sequence, exactly per the "Resolution direction agreed" below. The two documents no longer
describe the same event from two owners.

**The gap:** "Set up Scheduling" was meant to mean the timing *infrastructure*: read and
validate Quartz configuration, build and register Job/Trigger definitions, clean up orphaned
triggers, start the clustered scheduler, provide the window-calculation function as a
capability, and the admin controls. Instead, the current scheduling document claims ownership
of what happens *when a trigger fires* — "Scheduling has exactly one job: on a timetable, wake
up and say 'produce this report type, covering this time window'" — take the fire time, resolve
the window, create the Run, check the feature flag, hand off. That sequence is really the entry
point of the message production pipeline's own scheduled-trigger path, which already documents
nearly the same sequence independently in its own 1.7.A ("The scheduler fires... creates one
Run... checks a feature flag..."). The two documents currently describe the same event twice,
from two different owners, because the boundary was never cleanly drawn.

**Why it matters:** Concretely, it's what caused an earlier back-and-forth in this same project
about whether scheduling or the pipeline "really" creates the Run — a question that only exists
because the boundary was blurry. Cleanly separating "the machinery that wakes something up at
the right time" (scheduling) from "what happens once it does" (the pipeline's own scheduled
entry point, which calls into scheduling's window function as a shared utility) removes that
ambiguity for good, and stops the same event from being documented twice.

**Resolution direction agreed:**
- Scheduling owns: reading/validating Quartz config, building and registering Job/Trigger
  definitions, orphaned-trigger cleanup, starting the clustered scheduler, the window-calculation
  function (as a capability the pipeline calls into), and the admin control surface.
- The pipeline owns: the full sequence that runs when a scheduled trigger fires (take the fire
  time, resolve the window via scheduling's function, create the Run, check the feature flag,
  hand off/continue) — this is the entry point of its own already-documented "Scheduled" trigger
  path (1.7.A), not a separate "scheduling" concern.
- Open nuance, not yet decided: Run now and Backfill invoke that same fire-time logic on demand.
  Leaning toward keeping these in scheduling's domain (they're operator control over *whether/
  when* a firing happens, not over what it does), but this hasn't been finally settled.

**Affected documents:** Both `01-commander-scheduling.txt` and
`02-commander-message-production-pipeline.txt` need real restructuring, not just wording fixes —
this changes what each document claims ownership of, not merely how it's phrased.

---

## 11. [x] Direct contradiction: who creates the Run, and when, relative to the handoff — RESOLVED 2026-09-20

**Resolution:** Falls out of #10's fix. Only Assembly's 1.7 A claims Run creation now
("It creates a Run as its first durable action"); Scheduling's 1.7 A makes no competing claim.

**The gap:** This is a concrete symptom of #10, worth calling out on its own because it isn't
just duplicated documentation — the two documents actually disagree.

- Scheduling states the Run is created **before** the handoff, **by scheduling**: "The tracking
  record for the firing is created as the very first durable step, before this handoff happens"
  (1.6), and 1.7.A step 3, "It [the pod running scheduling's job] creates one tracking record
  (a 'Run') as its first durable action."
- The pipeline states the opposite: "This pipeline creates the Run for that firing and takes it
  from there" (1.6, Handoff boundaries) — implying the Run comes into existence as part of the
  pipeline's own processing, not before a scheduling-side handoff.

**Why it matters:** These can't both be true as currently worded. It's the same underlying
confusion as #10 (the fire→resolve-window→create-Run→check-flag sequence not having a single,
clear owner), surfacing here as an outright contradiction rather than mere duplication.

**Resolution:** Falls directly out of #10's agreed direction — the pipeline creates the Run, as
the first step of its own scheduled-trigger entry point. Scheduling's claims to the contrary
need to change, not the pipeline's.

**Affected documents:** Same as #10.

---

## 12. [x] Terminology inconsistency across the two documents — RESOLVED 2026-09-20

**Resolution:** All three sub-items resolved in `solution-document-commander.md`. (a) "The
watchdog" is now the sole name used throughout. (b) "Run" is the consistent name in mechanics
sections (1.6/1.7); 1.1 deliberately uses the generic "tracking record" instead — not a relapse,
but the intentional split between generic conceptual language and named mechanics established
elsewhere in this project. (c) The ambiguous "concurrent"/"overlapping" wording is gone from both
spots that used to collide — Scheduling's 1.7 C now says "no mutual-exclusion lock between
firings"; Assembly's 1.7 E now says "no mutual-exclusion lock on Runs for the same report type
and frequency" and "target the very same configuration and window on purpose" — distinct wording
for the two distinct concepts.

**The gap:** Three separate cases of the same thing being named or emphasized differently
depending on which document you're reading:

a. **"Watchdog" vs. "the pipeline's own recovery mechanism."** The pipeline document names this
   component consistently ("the watchdog," used throughout 1.4, 1.6, 1.7.D). The scheduling
   document never uses that name — it says "the pipeline's own recovery mechanism" (1.4 decision
   1, 1.7.C). Same component, two names, depending on which document you're in.
b. **"Tracking record" vs. "Run" — which one leads.** Scheduling treats "tracking record" as the
   primary, plain-language term (used throughout) and mentions "Run" once, parenthetically, as
   its technical name. The pipeline document does the opposite — "Run" is the primary term
   throughout, "tracking record" appears only twice, secondary. Same entity, inverted emphasis.
c. **"Concurrent" / "overlapping" describes two different things.** Scheduling's decision 8 and
   1.7.C use this language for two *consecutive* firings of the *same* (report type, frequency)
   overlapping in time (e.g., a 12:30 run still going when 13:00 fires). The pipeline's decision
   4 and 1.7.E use the same words for a *scheduled* run and an *on-demand* request targeting the
   *same window* both being allowed to proceed. Not contradictory, but similar wording for two
   genuinely different concepts risks a reader conflating them across documents.

**Why it matters:** Smaller than #10/#11, but the same root cause — content built up on both
sides of an unclear boundary, without a single shared vocabulary. Worth fixing in the same pass
rather than separately, since the new consolidated document is exactly where "one name per
concept" becomes easy to enforce.

**Affected documents:** Same as #10.

---

## 13. [x] On-demand guard against a PHT-only config — RESOLVED 2026-09-20

**Resolution:** Decided: defend inside Commander, not just at the source. Added as
`solution-document-commander.md`, 1.4 Architectural Decisions, decision 11 — a `NEVER`
(PHT-only) configuration supplied to the on-demand path is skipped and logged against the Run
rather than processed or failing the whole request. Reflected in the On-demand workflow steps
(1.7, Assembly A, step 3).

**The gap:** The on-demand path accepts an explicit list of configuration ids and does not
filter by frequency. A config marked `NEVER` (the PHT-only marker) could in principle be
included in an on-demand request by mistake. The product owner's guarantee is that this won't
happen — no on-demand request will ever target a PHT-only recipient — but whether Commander
should also guard against it in code, as a safety net in case that guarantee is ever wrong, is
still an open decision: enforce it at the source (the on-demand caller never sends one, since
the same team builds that caller), or defend inside Commander with a skip-and-log check that
records the skipped ids on the Run without failing the whole request.

**Why it matters:** A wrong assumption here wouldn't be caught by any existing mechanism —
Assembly has no reason to distinguish a `NEVER` config from any other on the on-demand path
today.

**Affected documents:** `solution-document-commander.md`, Assembly (1.5 Assumptions and/or 1.4
Architectural Decisions, once that section is built).

---

## 14. [x] `ProcessedInboundMessage` retention — RESOLVED (provisionally) 2026-09-20

**Resolution:** Added as `solution-document-commander.md`, 1.4 Architectural Decisions,
decision 12 — 24 hours, based on a provisional estimate (5 redelivery attempts × ~5-10 min pod
restart each) rather than the real queue config, which isn't known yet. Explicitly flagged in
the document as provisional and to be revisited once the actual backout-threshold and
redelivery configuration on the on-demand and PHT queues is confirmed.

**The gap:** How long to keep `ProcessedInboundMessage` rows isn't decided. It needs to be read
off the actual backout/redelivery-limit configuration on the on-demand and PHT queues, not
picked arbitrarily — a row swept before that window closes means a legitimate late redelivery
gets treated as new and reprocessed.

**Why it matters:** Too short a retention silently reopens the exact duplicate-processing risk
`ProcessedInboundMessage` exists to close; too long wastes storage for no benefit.

**Affected documents:** `solution-document-commander.md`, Assembly (1.5 Assumptions).

---

## 15. [ ] The watchdog job's own operational detail is unspecified

**The gap:** The watchdog's *behavior* is documented (1.6/1.7, Assembly) — what it does once it
finds a stale run — but not its own operational specifics: how often it actually polls, what
counts as "stale" (the heartbeat-timeout threshold), and whether either is configurable. Also
missing: the source material describes a **second, separate job** for on-demand/PHT Runs — a
low-frequency job that marks long-stale non-scheduled Runs `ABANDONED` for audit hygiene only,
explicitly *not* attempting recovery (since on-demand/PHT already recover via queue redelivery,
per 1.7 D). That second job isn't mentioned anywhere in the current document at all. Finally,
there's no admin surface for the watchdog itself — no way to check its health, see how many
runs currently look stale, or manually trigger a pass.

**Why it matters:** Same shape as #8 — a job that's named and partially behaviorally specified,
but not fully designed. An operator has no visibility into whether the watchdog is healthy, and
the separate on-demand/PHT audit-hygiene job doesn't exist in the document at all yet, even
though it's a real, distinct piece of the recovery story.

**Affected documents:** `solution-document-commander.md`, Assembly (1.4 Architectural Decisions
and/or 1.6 Logical View, once revisited). Explicitly deferred until Assembly is further
solutioned — noted 2026-09-19, not to be drafted prematurely.

---

## 16. [ ] ReportMessage structure suggestion

**The gap:** Distinct from #5 (which is about the *evolution strategy* — how the schema changes
safely over time), this is about the payload's actual *structure* — which has never been
proposed anywhere. The source material and 1.5 Assumptions only establish the identity/dedup
fields (`triggerType`, `configId`, `reportType`, `scopeKey`, `windowStart`, `windowEnd`,
`executionId`). Nothing addresses the data payload itself: what a Bundled request's merged
payment-type data looks like, what an Unbundled request's single-account payload looks like, or
whether CAMT.052 / CAMT.053 / CAMT.054 need different payload shapes given they carry
meaningfully different data (intraday balances, end-of-day statement data, debit/credit
notification data).

**Suggested starting structure** (for discussion, not yet a decision):

- **Envelope** — present on every request regardless of report type or bundling shape: the
  identity fields already established in 1.5 Assumptions (`triggerType`, `configId`,
  `reportType`, `scopeKey`, `windowStart`, `windowEnd`, `executionId`).
- **Body**, shaped by the bundling rule (1.7 B):
  - Bundled — a list of payment-type allocations, each carrying that type's accounts/aliases
    merged across every scope.
  - Unbundled — the single account or alias's own data.
  - Configuration-only — no scope-level data at all; just the configuration's own metadata.
- **Report-type-specific fields** — CAMT.052, CAMT.053, and CAMT.054 don't carry the same data
  (intraday balances vs. end-of-day statement data vs. debit/credit notification data), so the
  body likely needs a report-type-specific section rather than one shape fitting all three.

**Why it matters:** Executor needs a real payload contract to build against, not just the
identity fields used for dedup. Left unstated, whoever implements this first ends up choosing
the schema by accident rather than by decision.

**Affected documents:** `solution-document-commander.md`, most naturally landing once Section 2
(Implementation Reference) defines the actual message schema — but worth capturing the proposed
shape here now, before Executor's own contract gets built against something undocumented.

---

## 17. [ ] `ReportFrequency` code mismatch against the real schema, plus a possibly-missing constraint

**The gap:** Reading the real, pre-existing database init scripts
(`/home/vikunalabs/workspace/codespace/bikili/infra/docker/init-scripts/db/01-schema-reference.sql`
through `05-schema-report.sql`) surfaced a genuine mismatch against what's already documented.
The real `CAMT.ReportFrequency` seed data is: `EVERY_30_MIN`, `EVERY_1_HOUR`, `EVERY_2_HOURS`,
`EVERY_4_HOURS`, `DAILY`, `ONE_TIME_PER_DAY`, `FOUR_TIMES_PER_DAY`, `EIGHT_TIMES_PER_DAY`.
`commander-scheduling.md`'s Frequency catalogue instead uses `END_OF_DAY` (no `DAILY`
counterpart existed in the real data) and `NEVER` (the PHT-only marker — also absent from the
real seed data), and spells the once-a-day case `ONCE_PER_DAY` rather than the real
`ONE_TIME_PER_DAY`.

**Decided 2026-09-20:** `DAILY` = `END_OF_DAY` (the same concept, real code differs from ours —
`DAILY` is the value to use going forward). `NEVER` is a genuinely new value, not yet present in
the real schema, and needs to be added. `ONE_TIME_PER_DAY` vs. `ONCE_PER_DAY` still needs a
call — which spelling is authoritative.

**Also needed:** check whether any existing constraint on `CAMT.ReportConfig` or
`CAMT.ReportFrequency` needs adding or updating once `NEVER` is introduced as a real value —
not yet investigated. Candidates to check: whether `NEVER`-frequency configs need any special
handling beyond the structural exclusion already described (the scheduled path's own
`ReportType = ? AND ReportFrequency = ?` predicate never equals `NEVER`, per `goal.md`), and
whether `UX_ReportConfig_RecipientReportType UNIQUE (MessageRecipientId, ReportType)` has any
interaction with a recipient needing both a scheduled config and a `NEVER`/PHT config for
related report types.

**Why it matters:** `commander-scheduling.md`'s entire Frequency catalogue, generated-cron
table, and window-function examples are built around the code names currently in that document.
If the real schema's names differ, either the document or the schema (or both, via a migration)
needs to change before either is treated as final.

**Affected documents:** `commander-scheduling.md` (Frequency catalogue, 7.1 configuration
examples, 7.2 generated-cron table), `commander-data-retrieval.md` (wherever `ReportFrequency`
values are referenced). A new section covering the real schema will be added to the solution
documentation, but only after the user's own review — not drafted yet.

---

## Priority order (suggested)

1. JMS acknowledgment mode (correctness-critical, top priority)
2. Observability / correlation-ID propagation
3. Retention / archival policy
4. Secrets / credential management
5. Message contract / schema-evolution strategy
6. Rolling-deploy vs. crash
7. Sizing / throughput bounds
8. Pipeline-side admin surface
9. Recipient identity resolution (defer to data-retrieval document)
