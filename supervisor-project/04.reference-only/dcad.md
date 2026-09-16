# Solution Design Document - CAMT Report Generation - Commander

## Overview

This document describes the design of the CAMT Report Generation – Commander application. Its purpose is to enable the Corporate Reporting platform to produce ISO 20022 CAMT reports based on customer-specific report configurations, supporting both automated scheduled generation and customer-initiated on-demand generation.

Commander supports two processing paths:

**Scheduled Processing** — Commander triggers batch jobs at defined intervals based on report configurations stored in the CAMT database. For each job, it performs the following steps:

1. Reads all active report configurations that match the required report type and frequency, along with the associated message recipient information and corporate reporting agreement information where available.
2. Assembles each configuration into one or more report request messages (commands). Each message includes recipient details, agreement information, payment types, accounts, aliases, and other data that defines how the ISO 20022 CAMT report should be generated and what information it should contain.
3. Publishes each report request message to IBM MQ for consumption by the Executor application, which is responsible for the actual report generation.

**On-Demand Processing** — Commander also listens for customer-initiated report requests arriving on a dedicated IBM MQ queue. For each on-demand request, it validates the request against active report configurations and the Feature Flag service, then assembles and publishes a report request message through the same downstream pipeline used by scheduled processing. Requests that fail validation are rejected and recorded in the audit trail.

**Originator Onboarding Processing (Tactical Solution)** — Commander listens for originator onboarding events from an external system via a dedicated IBM MQ queue. When an originator is onboarded for Corporate Pay, Commander receive the onboarding message, validate it and invokes the CRAM (Corporate Reporting Agreement Management) API to synchronize the recipient configuration for CAMT.054D report type.

The system supports multiple schedule types, ranging from simple cron-based intervals to configurable intraday window-based patterns.

The main benefits of this solution are a fully automated, cluster-safe, and operationally configurable report dispatch pipeline. It guarantees exactly-once execution per trigger across a multi-node deployment, uses a parallel processing model that significantly reduces wall-clock time compared to sequential dispatch, and provides a unified processing path so that scheduled and on-demand requests are handled consistently once a report request message has been published to IBM MQ.

### 1.1 Conceptual Overview
The following high-level diagram illustrates the conceptual architecture of the solution:

<!-- Placeholder for diagram -->

**Commander** – Serves as the orchestration layer for both processing paths. For scheduled processing, it fires batch jobs at configured intervals, reads active report configurations from the CAMT ReportGen database, assembles report request messages, and publishes them to IBM MQ. For on-demand processing, it listens on a dedicated IBM MQ queue for customer-initiated requests, validates each request, and if valid — assembles and publishes a report request message through the same downstream pipeline.

**Executor** – Listens for report request messages from Commander via IBM MQ and performs data (payments and/or transactions) retrieval from the Operational Data Store (ODS), XML generation, XSD validation, and SFTP delivery of the final CAMT reports alongside a notification. Executor handles both scheduled and on-demand requests identically once the message arrives on the main processing queue.

### 1.2 Scope and Acceptance Criteria

#### 1.2.1 Scope

* Scheduling and triggering of batch jobs at configurable intervals, supporting both cron-based patterns (e.g., daily, every N hours) and intraday window-based patterns (e.g., N times per day at specific boundary times).
* Reading active report configurations from the CAMT SQL Server database, including report type, frequency, recipient information, and corporate reporting agreement data where available, using batch processing with snapshot isolation.
* Assembling each configuration into one or more report request messages according to configurable bundling rules (unbundled: one message per account/alias; bundled: one message per configuration).
* Publishing each report request message to IBM MQ for downstream processing by the Executor application. On publish failure, retrying with exponential backoff (up to 3 attempts) before routing to a dedicated retry queue.
* Persisting messages that exhaust all retry attempts to a dead letter table for manual operational intervention, with a background retry job that periodically re-processes pending dead letter entries.
* Executing multiple report type processors in parallel within a single job to reduce overall wall-clock time.
* Ensuring exactly-once trigger execution across a multi-node clustered deployment through a combination of Quartz JDBC JobStore cluster-safe scheduling and application-level deduplication via database unique constraints.
* Coordinating startup initialization across cluster nodes to prevent redundant job registration.
* Calculating reporting periods correctly for both cron-based lookback windows and boundary-to-boundary window-based schedules, including overnight windows that span midnight.
* Reflecting dynamic changes to report configurations (activation, modification, deactivation) in the next scheduled job cycle without requiring an application restart.
* Integrating with an external Feature Flag service (Harness) for per-report-type operational control, with fail-closed behavior when the service is unavailable.
* Supporting on-demand report requests received from customers via a dedicated IBM MQ On-Demand Queue, with configurable concurrency limiting. For each on-demand request, Commander validates that the requested report type is supported, that a matching active configuration exists in report configuration table within the CAMT database, and that the corresponding feature flag is enabled. If all validations pass, the request is assembled according to bundling rules? and published to the main processing queue. If any validation fails, no report request is published and a failure audit record is written with the rejection reason.
* Applying rate limiting on IBM MQ publish throughput to prevent thundering herd conditions.
* Exposing health check endpoints (liveness, readiness) and Prometheus metrics for operational monitoring, with defined alert thresholds.
* Maintaining an audit trail of all report request processing attempts in the report request audit table.

#### 1.2.2 Out of Scope

The following responsibilities are explicitly outside the boundary of the Commander application.

* Generation of ISO 20022 CAMT report files (performed by the Executor application).
* Fetching payment and/or transaction data from the Operational Datastore (performed by the Executor application).
* The On-Demand API — the external-facing HTTP API through which customers submit on-demand report requests. Commander consumes from the ONDEMAND.QUEUE only; it has no knowledge of or dependency on how requests arrive at that queue.
* Manual intervention and reprocessing of dead letter entries — this is an operational activity outside Commander.
* Delivery or transport of generated reports to end customers.
* Report archival, retention, or lifecycle management.
* User interface for report configuration management (configurations are managed externally).
* Transactional guarantees across multiple message publications (each message is published independently).

#### 1.2.3 Acceptance Criteria

* Trigger accuracy within ±60 seconds — A scheduled job MUST fire within ±60 seconds of its configured time under normal operating conditions.
* Exactly-once execution across cluster — Across a multi-node cluster, each scheduled trigger MUST execute on exactly one node. No duplicate executions and no missed executions; deduplication constraint prevents duplicate audit entries
* Period calculation accuracy for window-based schedules — For a window-based schedule with configured boundaries (e.g., 10:00, 13:00, 18:00, 21:00), the job firing at 13:00 MUST calculate its reporting period as [10:00, 13:00] on the same calendar day.
* Configuration pickup without restart — When a report configuration is activated, modified, or deactivated in the database, the next scheduled job execution for that report type MUST reflect the change without application restart.
* Bundling correctness (unbundled = one message per account/alias; bundled = one message per configuration) — For a configuration marked as unbundled, one message per account (or alias) across all agreement scopes MUST be published. For a bundled configuration, exactly one report message per configuration MUST be published.
* Parallel processing — For a schedule type supporting multiple report types, all eligible processors MUST execute concurrently. Total job wall-clock time MUST approximate the duration of the longest-running processor, not the sum of all processors.
* Failure isolation — Failure to publish one report message MUST NOT block publication of remaining messages in the same batch. The job MUST continue processing all configurations regardless of individual publish failures.
* Consolidated failure reporting — After all messages in a batch are processed, any failures that could not be routed to a retry channel MUST be logged as a consolidated exception. The job itself MUST NOT be marked as failed by the scheduler.
* Cluster startup safety — When multiple nodes start simultaneously, exactly one node MUST execute job registration. Other nodes MUST skip registration. The registration operation itself MUST be idempotent.
* Misfire handling — When a trigger misfires due to a previous execution still running, the scheduler MUST fire the missed trigger once immediately after the current execution completes, then resume the normal schedule.
* Operational visibility — Every job execution MUST produce logs containing: schedule type, fire time, number of configurations processed, number of messages published, number of failures (if any), and total execution duration.
* Timezone consistency — All schedules MUST respect the configured time zone (Europe/Stockholm), including correct handling of daylight saving transitions.
* On-demand request rejected when report type is not supported — Unsupported report type returns rejection; no message published to processing queue; failure audit record written with rejection reason.
* On-demand request rejected when no active configuration exists — Valid report type but no matching active `report_config` row returns rejection; no message published; failure audit record written.
* On-demand request rejected when feature flag is disabled — Valid report type and active config but feature flag disabled returns rejection; no message published; failure audit record written.
* On-demand audit trail written on both success and failure paths — Successful and rejected on-demand requests both produce a record in the report request audit table with the correct status and reason.
* Dead letter retry processes pending messages within X minutes of failure. Monitor `dead_letter_requests` table; verify background job execution timing.
* Graceful shutdown completes in-flight jobs within 30 seconds
* Metric availability — All core metrics (job execution count, duration, message publication success/failure, queue depths) available in Prometheus and Grafana dashboards

### 1.3 Information Relevant for Prioritization

Provide or update information so that this feature can be included in a full WSJF prioritization session in the current Program Increment (PI).  This should include information relevant for

Opportunity Enablement / Risk Reduction (OE/RR)
Time Cruciality (TC)
Business Value (BV)
Job Duration (Size)

### 1.4 Architectural decisions

Provides the BC of an acceptable solution

| Date | Decision | Name |
|:----:|:----:|:----:|
|NA|NA|NA|

### 1.5 Assumptions and pre-requisites

From a business architectural point-of-view provide assumptions and pre-requisites required for this feature.  For example:

* Amazon cloud (AWS) used to deploy the solution
* Legal agreement with AWS and Infosys concluded
* E-enablers such as login, security, e-archive, signing and e-signing are used for the solution
* Same interface for customer facing employees and customers with more or less some functionality with differences based on roles and entitlement configuration.
* Use PKI when applicable

### 1.6 Logical view – architecture scope

Here is where the overview of the architecture in logical components are presented.

The depiction here is to determine which logical components that compose the solution and how they interact.

#### 1.6.1 Overview of the logical architecture

The Commander application consists of four logical components that collaborate during each job execution:

| Component | Responsibility |
|-----------|----------------|
|Scheduler Engine|Triggers jobs at configured times; persists schedule state across restarts; coordinates across cluster nodes to ensure exactly-once trigger execution|
|Job Orchestrator|Executes when a trigger fires; determines the reporting period; spawns concurrent processors for each eligible report type|
|Configuration Reader|Reads active report configurations from the CAMT ReportGen database for the relevant report type and frequency|
|Message Publisher|Assembles report request messages from configuration data and publishes them to the downstream processing queue|
|On-Demand Request Handler| Consumes customer-initiated report requests from the On-Demand Queue; validates each request and routes valid requests into the main processing pipeline|


The interaction between these components during a scheduled execution is shown below:

<!-- Placeholder for diagram -->

Note: Each Processor independently invokes the Configuration Reader and Message Publisher. The arrows from multiple Processors to Configuration Reader represent independent, concurrent calls — not a single shared invocation.

#### 1.6.2 Scheduling Patterns

The solution addresses two distinct scheduling patterns required by different CAMT report types:

**Pattern A: Fixed Clock-Aligned Intervals**

This pattern is designed for reports that must be generated at consistent, evenly spaced times throughout the day. Examples include:

* End-of-Day Reports: Generated daily at midnight, such as CAMT053 and CAMT054
* Intraday Reports: Generated at uniform intervals such as CAMT052

    * Every 30 minutes
    * Every 1 hour
    * Every 2 hours
    * Every 4 hours

For these schedules, the reporting period is a lookback window ending at the fire time. For instance, a job firing at 04:00 on a 4-hour schedule produces the period [00:00, 04:00] — capturing all activity from midnight to 4 AM.

**Pattern B: Configurable Intraday Windows**

This pattern addresses cases where reports need to be generated at specific, user-defined times that align with business operations or irregular schedules. For instance, a customer may configure reporting times at 10:00, 13:00, 18:00, and 21:00 — not evenly spaced, but aligned to business cut-off times.

For these schedules, the reporting period is the interval between consecutive boundaries. The 13:00 job produces [10:00, 13:00]; the 18:00 job produces [13:00, 18:00]. The period cannot be derived by arithmetic on the fire time alone — it requires knowledge of the preceding configured boundary.

#### 1.6.3 Scheduler Types and Reporting Mapping

The solution defines eight schedule types. Each schedule type corresponds to a frequency value stored in the report configuration table, and each is associated with one or more CAMT report types:

| Schedule Type       | Strategy         | Fire Pattern        | Associated Report Types |
|---------------------|------------------|---------------------|--------------------------|
|DAILY_MIDNIGHT|	Fixed interval|	Daily at 00:00	|CAMT.053 Standard, CAMT.053 Extended, CAMT.054 Debit Notification|
|EVERY_30_MIN	|Fixed interval	|Every 30 minutes	|CAMT.052 Balance, CAMT.052 Balance & Transactions|
|EVERY_1_HOUR	|Fixed interval	|Every hour at :00	|CAMT.052 Balance, CAMT.052 Balance & Transactions|
|EVERY_2_HOURS	|Fixed interval	|00:00, 02:00, … 22:00	|CAMT.052 Balance, CAMT.052 Balance & Transactions|
|EVERY_4_HOURS	|Fixed interval	|00:00, 04:00, … 20:00	|CAMT.052 Balance, CAMT.052 Balance & Transactions|
|ONE_TIME_PER_DAY|	Intraday window	|1 configured boundary|	CAMT.054 Credit Notification|
|FOUR_TIMES_PER_DAY|	Intraday window|	4 configured boundary|	CAMT.054 Credit Notification|
|EIGHT_TIMES_PER_DAY|	Intraday window	|8 configured boundary	|CAMT.054 Credit Notification|

#### 1.6.4 Reporting Period Calculation

The reporting period for each job execution is determined by the schedule type and the time the trigger was scheduled to fire.

**Fixed Interval Schedules:**

The period is a lookback window of exactly one interval, ending at the scheduled fire time: 

|Schedule Type | Reporting Period |
|----|----|
|DAILY_MIDNIGHT|	Previous calendar day 00:00 to 00:00 |
|EVERY_30_MIN	| [fire time − 30 min, fire time] |
|EVERY_1_HOUR	| [fire time − 1 hour, fire time] |
|EVERY_2_HOURS	| [fire time − 2 hours, fire time] |
|EVERY_4_HOURS	| [fire time − 2 hours, fire time] |

**Intraday Window Schedules**

The period is the interval between the previous configured boundary and the current one:

|Trigger |	Fires At |	Reporting Period (example: boundaries 10:00, 13:00, 18:00, 21:00) |
|----|----|----|
|Window 1|	10:00 |	[00:00, 10:00]|
|Window 2|	13:00 |	[10:00, 13:00]|
|Window 3|	18:00 |	[13:00, 18:00]|
|Window 4|	21:00 |	[18:00, 21:00]|

**Timezone handling**: Period boundaries are calculated in the Europe/Stockholm timezone, then converted to UTC for storage and all downstream communication. This ensures that schedule definitions remain aligned with business expectations while all data exchange is unambiguous and timezone-independent.

During daylight saving transitions, the same Europe/Stockholm local window produces different UTC ranges — this is expected and correct behavior.

#### 1.6.5 End-to-End Processing Flow

The sequence below describes a complete execution from trigger through message publication.

**Scheduled Processing**

1. The Scheduler Engine fires a trigger at the configured time and assigns execution to one cluster node.
2. The Job Orchestrator calculates the reporting period for this trigger and identifies which report types are eligible.
3. For each eligible report type, a Processor is started concurrently.
4. Each Processor reads active report configurations from the CAMT database, assembles report request messages following bundling rules, and publishes each message to the main processing queue.
5. On publish failure, the message is retried; if retries are exhausted, it is routed to the retry queue or dead letter table.
6. Once all Processors complete, the Job Orchestrator logs a consolidated summary including the count of configurations processed, messages published, and any failures.

**On-Demand Processing**

1. A customer submits a report request via the On-Demand API (external to Commander), which places the request on the On-Demand Queue.
2. The On-Demand Request Handler consumes the request and validates it against active report configurations and the Feature Flag service.
3. If validation passes, the request is assembled and published to the main processing queue — the same queue used by scheduled processing.
4. If validation fails, no message is published and a failure audit record is written with the rejection reason.

**Downstream Processing (Executor)**

Once a report request message is published to the main processing queue, the Executor application takes over. It retrieves transaction and payment data from the Operational DataStore, generates the ISO 20022 CAMT XML report, validates it against the schema, and delivers it to the customer's SFTP destination.

### 1.7 Report Configuration

#### 1.7.1 Configuration Model

Each report configuration represents a customer-specific instruction for how and when a particular CAMT report should be generated. A configuration captures:

#### 1.7.2 Bundling Rules

The bundling mode controls how report request messages are assembled from a single configuration:

| Mode |	Behavior |	Typical Use Case |
|-----|-----|------|
| Unbundled | One report request message per account or alias	| Each account (or alias) is processed independently, allowing individual delivery |
| Bundled |	One report request message covering all accounts and/or alias within the configuration	| Consolidated report for a single customer across all their accounts and/or alias |

Example: A configuration covering three accounts produces either three messages (unbundled) or one message (bundled).

### 1.8 Message Queue Resilience and Failure Handling

#### 1.8.1 Three-Tier Failure Handling

Commander uses a three-tier approach to ensure report request messages are not silently lost when message queue publication fails:

<!-- Placeholder for diagram -->

Tier 1 — Exponential backoff retry: On a publish failure, Commander retries the same message up to three times with increasing delays between attempts. This handles transient connectivity issues without immediately escalating.

Tier 2 — Retry queue: If all retry attempts are exhausted, the message is routed to a dedicated retry queue for deferred processing by a separate retry consumer.

Tier 3 — Dead letter table: If the retry queue is also unavailable, the message payload is persisted to a dead letter table in the database. A background job runs periodically to re-attempt delivery for entries in this table. After a configurable number of reprocess failures, the entry is marked as a permanent failure and an operational alert is triggered.

#### 1.8.2 Error Classification

Not all failures are treated equally. Commander distinguishes between two categories:

|Error Type	| Examples|	Handling|
|-----|-----|------|
|Transient	|MQ connectivity timeout, temporary network interruption|	Retry with exponential backoff|
|Permanent	|Invalid configuration, unrecognized message format	|Route directly to dead letter — no retries|

This prevents endless retry loops for failures that will never self-resolve.

#### 1.8.3 Rate Limiting

To prevent thundering herd conditions during peak schedule windows — where many report types fire simultaneously — Commander applies a configurable rate limit on message publication throughput. This protects the message broker from sudden spikes in inbound traffic.

#### 1.8.4 Failure Isolation

A failure to publish one report message does not block publication of remaining messages in the same batch. Each message is processed independently. At the end of the job, a consolidated summary is logged capturing the total count of published messages, failed messages, and any messages routed to the retry queue or dead letter table.

### 1.9 Feature Flag Integration

#### 1.9.1 Purpose

Commander integrates with an external Feature Flag service (Harness) to provide operational control over which CAMT report types are active at any given time. This allows individual report types to be enabled or disabled instantly — without a deployment — by toggling a flag in the Feature Flag service.

Feature flags are evaluated for both scheduled and on-demand processing paths. If a flag is disabled for a report type, no report request message is published for that type, regardless of how many active configurations exist.

#### 1.9.2 Behavior

| Scenario | Behavior |
|----------|----------|
| Flag enabled | Report type proceeds normally |
| Flag disabled | Report type skipped; audit record written with `SKIPPED` status |
| Flag service unreachable, cached value available | Cached value is used |
| Flag service unreachable, no cached value | Fail-closed: report type is skipped |

The fail-closed policy is intentional: in the absence of a definitive enabled signal, Commander errors on the side of not generating reports rather than generating potentially unwanted output.

#### 1.9.3 Caching

Flag values are cached for a short, configurable TTL (default: X seconds) to reduce latency and avoid overloading the Feature Flag service during high-frequency schedule windows. The cache is refreshed on next access after expiry.

### 1.10 Batch Processing

#### 1.10.1 Why Batch Processing Is Required

At scale, a single frequency bucket can contain tens of thousands of active report configurations. Loading all configurations into memory at once would risk out-of-memory conditions and degrade performance. Commander addresses this through a batched processing model.

#### 1.10.2 Processing Approach

To be completed

### 1.11 Audit and Deduplication

#### 1.11.1 Purpose

Commander maintains an audit trail of every report request processing attempt. This serves two purposes:

**Operational visibility** — a queryable record of what was published, when, for which message recipient, and with what outcome.
**Deduplication** — a unique constraint on the audit table prevents the same report request from being published more than once for the same combination of report type, message recipient, business date, and frequency.

#### 1.11.2 Audit Record Lifecycle

Each processing attempt produces an audit record with one of the following outcomes:

|Outcome	|Meaning|
|-----|-----|
|PUBLISHED|	Report request message successfully published to the main processing queue|
|SKIPPED|	Processing skipped due to feature flag disabled or configuration deactivated|
|FAILED	|Publishing failed after exhausting all retry attempts|
|DUPLICATE|	Unique constraint violation detected — request already processed for this combination|

#### 1.11.3 Deduplication Mechanism

Before publishing, Commander attempts to insert a `PENDING` audit record. If the insert fails due to a unique constraint violation, the request is identified as a duplicate and processing is skipped. This approach provides atomic, lock-free deduplication without requiring distributed coordination.

#### 1.11.4 Retention

Audit records are retained in the primary table for X (90?) days. Older records are archived for long-term retention (X years) in a separate partitioned table to support regulatory and operational audit requirements.

### 1.12 Observability

#### 1.12.1 Structured Logging

Every job execution produces a structured log entry at completion, containing:

|Field	| Description|
|-----|-----|
|Schedule type|	The frequency type that fired (e.g., `FOUR_TIMES_PER_DAY`)|
|Fire time	|The time the trigger was scheduled to fire (UTC)|
|Reporting period|	The start and end of the calculated reporting window (UTC)|
|Configurations processed	|Total number of active configurations evaluated|
|Messages published	|Total number of report request messages successfully published|
|Messages failed	|Count of messages that could not be published after retries|
|Duration|	Total wall-clock execution time for the job|

All logs are structured JSON and include a correlation ID, instance identifier, and trace ID for distributed tracing across Commander and Executor.

#### 1.12.2 Metrics

Commander exposes operational metrics via a Prometheus-compatible endpoint, covering:

|Category	|What Is Measured|
|-----|-----|
|Job execution	|Total executions by schedule type; outcome (success/failure); duration|
|Message publishing	|Total messages published by report type and queue; publish error count|
|Feature flags	|Current enabled/disabled status per report type; API call outcomes|
|Batch processing|	Duration per batch|
|Dead letter	|Current count of entries by status|
|Active configurations|	Current count of active configurations per frequency|

#### 1.12.3 Health Check Endpoints

Commander exposes two health check endpoints for use by the platform orchestration layer:

|Endpoint|	Type|	Returns healthy when|
|-----|-----|-----|
|/health/live|	Liveness	|The application process is running|
|/health/ready|	Readiness	|Database is reachable, message broker is connected, Feature Flag service is reachable, and the scheduler is initialised|
