# Requirements

## Definitions

**Report Message Execution** — A single logical run of the report generation workflow, triggered either by a Quartz schedule firing for a specific report type and frequency, or by an accepted on-demand request. A report message execution covers the full span of work from the moment it's triggered through to every `ReportConfig` in its scope being processed and every resulting report message being published (or terminally failed).

Each report message execution has its own unique, persistent identifier that is assigned when the execution begins and stays the same across any retry or recovery — it is never reassigned, even if the execution is interrupted and resumed by a different pod.

A report message execution may produce many report messages (one or more per `ReportConfig` in scope, depending on bundling), but it is itself a single unit — a report message execution is either still in progress, complete, or has failed; it does not have a partial identity the way an individual message or record does.

A scheduled report message execution and an on-demand report message execution are the same concept applied to two different triggers, and are tracked independently of one another (per NFR11) — but both are "a report message execution" in this document's sense.

---

## 1. Business Requirements

**1.1 Scheduled Report Generation**
- **BR1** The application must generate and publish report messages on a recurring schedule, per report type, according to that report type's configured frequency (e.g., CAMT052B every 30 minutes, CAMT053S daily, CAMT054C up to eight times a day).
- **BR2** Only active report configurations should be processed for a given report type and frequency combination.

**1.2 Data Resolution and Message Content**
- **BR3** For each report configuration being processed, if it has an associated agreement scope, the report message must be built using the full resolved account/alias/payment-type data under that scope. If no such association exists, the report message must be built using the report configuration's own data alone.
- **BR4** Every configuration property on a report configuration (active status, bundling flag, account format, pagination flag, report type, report version, frequency, start/end date or datetime, etc.) must be included in the report message sent to the downstream application, regardless of whether that property affects processing logic.
- **BR5** Message fan-out must follow the bundling flag: bundled configurations produce a single message covering all associated accounts/aliases for a payment type; unbundled configurations produce one message per account/alias.
- **BR6** If a bundled message fails partway through generation, it must be regenerated in full on retry — never partially completed or partially missing accounts.
- **BR7** It is acceptable for a report message execution to reflect data as of when it runs; a configuration or association change that happens mid-run does not need to be reflected in that same run. Picking it up on the next scheduled run is sufficient.

**1.3 Message Delivery Guarantees**
- **BR8** Exactly-once publication to IBM MQ is a hard business requirement — a given logical report message must never be published more than once.
- **BR9** No report configuration may be treated as successfully processed unless its corresponding message(s) have actually been published.

**1.4 On-Demand Reports**
- **BR10** Users must be able to request a report on demand via an API, specifying recipient type, recipient value, report type, report version, and start/end date or datetime as applicable.
- **BR11** On-demand requests must go through the same data resolution and message generation rules as scheduled report message executions, including bundling behavior.
- **BR12** Acceptance of an on-demand request by the API is not the same as successful generation and publication; the request must be trackable through to completion independently.
- **BR13** An on-demand report message execution must be isolated from any concurrent scheduled report message execution touching overlapping data — the two must not run against the same scope at the same time.
- **BR14** A user resubmitting an identical on-demand request is making a new, independent request — not a duplicate to be suppressed at the API layer.
- **BR15** On-demand requests are expected to be picked up and processed promptly after submission, though no fixed SLA has been set.

**1.5 Resilience**
- **BR16** A report message execution that is interrupted must not be silently lost or require reprocessing everything from the beginning once the application recovers.
- **BR17** Recovery of an interrupted report message execution must not be attempted by more than one instance of the application at the same time.

---

## 2. Environmental Constraints

- **EC1** The application is a Spring Boot service.
- **EC2** Scheduling is handled by Quartz, configured with a clustered `JDBCJobStore` (`isClustered=true`) backed by the shared database. Only one pod in the cluster fires a given trigger at a time. Definition and management of individual triggers is a separate concern from this workflow.
- **EC3** The application is deployed on OpenShift, running as multiple concurrent pods. Any pod may be terminated unexpectedly (node eviction, resource limits, rolling deployment, crash).
- **EC4** Messages are published to IBM MQ via JMS. IBM MQ supports messages up to 100 MB — not a practical constraint at current data volumes.
- **EC5** The database is SQL Server, with the schema already defined under the `CAMT` schema (`ReportConfig`, `ReportAgreementScope`, `AgreementScope`, `PaymentTypeAssignment`, `AccountAssignment`, `AliasAssignment`, `Recipient`, `Agreement`, `AgreementVersion`, `ReportType`, `ReportTypeFrequency`, `PaymentType`).
- **EC6** A `ReportConfig` record has both an internal identity column (`Id`) and a separate business identifier (`ConfigId`), which is required to be non-null whenever the configuration is active.
- **EC7** A `ReportConfig` is uniquely constrained per recipient and report type — a given recipient can have at most one configuration per report type.
- **EC8** A single report message execution may process anywhere from roughly 1,000 to over 10,000 `ReportConfig` records.

---

## 3. Derived / Non-Functional Requirements

**NFR1 — Every report message execution needs a stable, persistent ID that survives a crash and a pod swap.**
*(from BR16, EC3)*
If a pod dies partway through a run and a different pod picks up recovery, that second pod needs some way to know it's looking at the same execution the first one started — not a new one. That has to be assigned up front and written somewhere every pod can see, since the pod that started the job may not exist anymore by the time recovery happens.

**NFR2 — Every report message needs a deterministic ID based on its business content, not on the run that produced it.**
*(from BR8, BR9, BR16)*
Exactly-once delivery only works if the system can look at a message it's about to send and ask "have I sent this before?" That's unanswerable if the message's ID changes every time it's regenerated on retry. The ID has to come from what the message represents — which account, which payment type, which report message execution — not from which attempt produced it.

**NFR3 — Execution state and checkpoints live in the shared database, never on the pod itself.**
*(from EC3, BR16, BR17)*
Pods come and go on OpenShift. If progress were tracked in memory or on local disk, it would disappear the moment the pod tracking it does, and there'd be nothing left to recover from. This also underpins BR17: coordinating who's allowed to recover a report message execution only works if the state describing that execution is visible to every pod, not private to one.

**NFR4 — The application has to notice an interrupted run on its own, not wait for the next Quartz trigger.**
*(from BR16, EC3)*
Some of these report types only run once a day. If detection depended on the next scheduled trigger, a report message execution that dies at 2 AM could sit unresolved for close to a day before anything notices. Detection needs to happen at startup, independent of the schedule.

**NFR5 — Resume from the last checkpoint; don't start the whole run over.**
*(from BR16, EC8)*
With executions running into the thousands or tens of thousands of records, each needing several downstream lookups, restarting from record one every time a pod gets killed would be expensive — and pod termination is routine here, not exceptional. Checkpointing caps the damage from any one interruption to roughly the last unit of work in progress, not the whole run.

**NFR6 — Recovery needs to tell "worth retrying" apart from "won't fix itself."**
*(from BR16)*
A report message execution that failed because the database blipped is worth retrying automatically. One that failed because the underlying data was bad will fail the same way every time — retrying it endlessly doesn't help and can quietly bury a data issue that needs a person to look at it. Recovery that treats every failure identically isn't really doing what BR16 intends.

**NFR7 — Only one pod may recover a given report message execution at a time.**
*(from BR17)*
Mostly BR17 restated in system terms — worth calling out because it doesn't happen automatically. With several pods starting up around the same time, more than one could spot the same interrupted execution and try to recover it unless something explicitly arbitrates.

**NFR8 — Only one pod may run a given scheduled trigger.**
*(from EC2)*
This is already satisfied — Quartz's clustered `JDBCJobStore` guarantees only one node in the cluster fires a given trigger. No additional mechanism is needed here.

**NFR9 — Resolving the agreement-scope hierarchy can't cost one query set per record.**
*(from EC8, BR3)*
BR3 asks for the full hierarchy to be resolved for every applicable configuration, and at these volumes, doing that one record at a time turns into tens of thousands of queries per run. This exists purely because BR3's data need and the volume in EC8 collide, not because query efficiency was asked for directly.

**NFR10 — Database state and MQ publication state need to be reconciled deliberately, not left to chance.**
*(from BR8, BR9)*
An ordinary "commit to the database, then send to MQ" sequence has a gap in the middle. A crash right after the commit but before the send leaves a configuration marked done with no message ever sent. A crash the other way around leaves a message sent with the database saying otherwise. Either one breaks BR8 or BR9, so something has to close that gap deliberately.

**NFR11 — On-demand tracking has to stand on its own, separate from scheduled report message execution tracking.**
*(from BR12, BR13)*
BR12 wants an on-demand request followed through to completion independently of the API call that started it, and BR13 wants on-demand and scheduled report message executions kept from stepping on each other's data. Both get harder if on-demand and scheduled executions share the same tracking state. Keeping them separate is what makes each one possible to reason about on its own.

**NFR12 — Something has to actively stop on-demand and scheduled report message executions from overlapping on the same data.**
*(from BR13)*
Mostly a restatement of BR13, called out because isolation doesn't happen by accident. With multiple pods and two independent entry points — Quartz on one side, the on-demand queue on the other — there's no natural reason a scheduled run and an on-demand request wouldn't land on the same scope at the same moment unless something is specifically built to prevent it.

**NFR13 — Bundle generation is all-or-nothing.**
*(from BR6)*
BR6, phrased as a system constraint rather than a business outcome. The business side is simple — never send a half-finished bundle. Guaranteeing that means the transaction or chunk boundary around building a bundle has to match the bundle boundary itself.

---

## 4. Architectural Decision Points

- **D1** Whether the processing workflow is implemented using Spring Batch or a plain Quartz-triggered Spring Boot service.
- **D2** The strategy for resolving the `ReportConfig → AgreementScope` hierarchy efficiently at volume (NFR9).
- **D3** The mechanism for detecting and recovering interrupted report message executions after a pod restart (NFR4, NFR5).
- **D4** The mechanism for coordinating recovery ownership across multiple pods, so recovery isn't attempted redundantly (NFR7).
- **D5** The pattern used to reconcile database and IBM MQ consistency to satisfy exactly-once publication (NFR10) — XA/JTA, transactional outbox, or another pattern.
- **D6** The chunking/unit-of-work size, trading off commit overhead against the size of the reprocessing window on failure (NFR5).
- **D7** The failure-handling and retry strategy for messages that can't be published (NFR6).
- **D8** How on-demand and scheduled report message executions are mutually isolated against overlapping data (NFR12).
- **D9** How the `ReportConfig.ConfigId` vs. `Id` distinction (EC6) maps onto the execution/message identifiers required by NFR1/NFR2 — i.e., whether `ConfigId` is the natural business key those identifiers should be built from.

---
