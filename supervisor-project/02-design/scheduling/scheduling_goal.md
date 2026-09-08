# Commander — Scheduling: Problem Brief

I want fresh approaches to how Commander's **scheduled** trigger is designed. Please propose
**2–3 distinct ways to solve this**, with trade-offs. This brief states the problem and the
constraints; it does not prescribe a design.

Scheduling is a separate concern from the message-production pipeline (that pipeline is
already designed — see `how-it-works.md` / `solutions_v08.md`). This brief is only about
deciding **when each report runs and what time window that run represents**.

---

## What a firing must produce

Per firing, scheduling hands the pipeline exactly:

> `report_type` (one) · `frequency` · `scheduled_time` · `window_start` · `window_end`

The pipeline then creates **one `Run`** and pages
`ReportConfig WHERE report_type = ? AND frequency = ? AND is_active = 1`. A `Run` is
**single-report-type** — that is a fixed property of the pipeline schema, not negotiable here.

---

## The cadences (business requirements)

A `ReportConfig` carries a `frequency` column; that value decides which schedule picks it up.
All times are in one configured business timezone. Business day = Mon–Fri, no holiday calendar.

| Report type(s) | `frequency` | Fires | Days | Reporting window |
|---|---|---|---|---|
| CAMT052B, CAMT052BT | `EVERY_30_MIN` | every 30 min, 00:30 → 21:00 | Mon–Fri | from the previous firing's time to this firing's time; the **first firing of the day covers from 00:00** |
| CAMT052B, CAMT052BT | `EVERY_1_HOUR` | hourly, 01:00 → 21:00 | Mon–Fri | ″ |
| CAMT052B, CAMT052BT | `EVERY_2_HOURS` | 03:00, 05:00 … 21:00 | Mon–Fri | ″ (first window 00:00 → 03:00) |
| CAMT052B, CAMT052BT | `EVERY_4_HOURS` | 05:00, 09:00, 13:00, 17:00, 21:00 | Mon–Fri | ″ (first window 00:00 → 05:00) |
| CAMT054C | `ONCE_PER_DAY` | 21:00 | Mon–Fri | 00:00 → 21:00, same day |
| CAMT054C | `FOUR_TIMES_PER_DAY` | 10:00, 13:00, 18:00, 21:00 | Mon–Fri | previous firing → this firing; first firing from 00:00 |
| CAMT054C | `EIGHT_TIMES_PER_DAY` | 8 configurable times, last = 21:00 | Mon–Fri | ″ |
| CAMT053S, CAMT053E, CAMT054D | `END_OF_DAY` | 06:00 | Tue–Sat | the **whole previous calendar day**, 00:00 → 24:00 |
| — | `NEVER` | never scheduled | — | — (PHT-only configs; produced only by an external push) |

Two window shapes: "interval / boundary-to-boundary within a day, anchored to midnight for the
first firing" (everything except `END_OF_DAY`), and "the entire previous calendar day"
(`END_OF_DAY` only).

---

## Fixed constraints

- **Spring Boot**, **Quartz** with a **clustered JDBC JobStore in SQL Server** (already the
  pipeline's setup). One shared database, one MQ manager.
- **Multiple identical pods.** Any pod can be terminated mid-work with no clean shutdown.
  Clustered Quartz guarantees only one pod fires a given trigger.
- **Schedules are fixed at deploy time** (properties / config), changed by redeploy. No
  runtime schedule editing and no schedule-management UI is in scope.
- All time arithmetic in **one configured business timezone**.

---

## Requirements the approach must satisfy

1. **Window from the scheduled fire time, never wall-clock.** A firing that runs late (misfire,
   pod restart, recovery) must still produce the window it was *scheduled* for.
2. **One firing → one `Run` per report type.** A schedule that covers several report types
   (e.g. the three `END_OF_DAY` reports) must result in independent `Run`s — separate progress,
   separate recovery.
3. **Misfire behaviour must be expressible.** If every pod was down across a firing, on
   recovery the missed slot should fire **once** — not replay every missed slot; older missed
   slots are left to a manual on-demand backfill. (The exact policy is being decided
   separately; the design just needs to make it configurable and keep the catch-up firing's
   window correct.)
4. **DST correctness** (this is financial reporting). Define behaviour when a boundary time
   does not exist (spring-forward gap) or occurs twice (fall-back overlap). Note `END_OF_DAY`'s
   "previous calendar day" legitimately spans 23 or 25 elapsed hours on a transition day.
5. **Idempotency-friendly.** The pipeline deduplicates on
   `(trigger_type, config_id, report_type, scope_key, window_start, window_end, execution_id)`.
   A catch-up firing or a repeated firing that represents the same logical window should
   collapse cleanly against an existing one, not double-produce.
6. **Startup validation.** Every `(report_type, frequency)` present in `ReportConfig` (other
   than `NEVER`) must have a matching schedule, or those configs would silently never be
   produced.
7. **Config maintainability.** Avoid two independently-authored sources of truth for the same
   fire times (e.g. a cron string *and* a boundary list) that can drift apart.
8. **Fast firing in TEST.** Non-production environments need a way to make a schedule fire far
   more often than production, to exercise the message flow quickly — ideally producing a
   fresh window (and fresh messages) on each firing. This is straightforward for the
   interval/boundary shapes but awkward for `END_OF_DAY`, whose window is a function of the
   date alone.

---

## Optional (minimal, may be dropped from v1)

- **Pause / Resume** a schedule at runtime without a redeploy. Keep minimal; resume
  forward-only (no backfill of firings missed while paused).

---

## Out of scope

- Producing the report messages (that is the pipeline).
- Schedule-management UI, runtime schedule CRUD, per-config schedule overrides.
- Holiday calendars.
