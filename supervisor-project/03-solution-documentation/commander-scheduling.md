# Commander — Scheduling (Infrastructure Setup)

## Overview

This document covers Commander's Scheduling component: the infrastructure that reads, validates, and runs the timetable driving Assembly's scheduled entry point. It covers startup (reading and validating configuration, building and registering the underlying jobs and triggers), the window-calculation function, and the admin control surface.

It does not cover what happens when a trigger fires — that is Assembly's responsibility, described in the main Commander solution document (`solution-document-commander.md`).

## 1.1 Conceptual Overview

Scheduling is Commander's timing infrastructure. It reads and validates the deploy-time schedule configuration, builds and registers the underlying Quartz jobs and triggers, keeps that registration clean across redeploys, and runs the clustered scheduler that fires them. It also exposes a window-calculation function, and an admin control surface for operators.

Scheduling's involvement ends the moment a trigger fires: at that point, it invokes Assembly's scheduled entry point and steps away. It does not decide what a report should contain, does not create any tracking record, and does not know whether a firing eventually succeeds — see the main Commander solution document for Assembly, which owns everything from that point on.

## 1.2 Scope and Acceptance Criteria

## 1.3 Information Relevant for Prioritization

## 1.4 Architectural Decisions

## 1.5 Assumptions and Pre-Requisites

- The business day is Monday to Friday. There is no holiday calendar — a weekday holiday still runs its scheduled firings and produces a (possibly near-empty) report.

## 1.6 Logical View - Architecture Scope

### Logical components

- **Trigger loader / generator** — reads the deploy-time schedule configuration, expands any entry that lists several report types into one independent schedule per report type, generates the underlying cron timing expression(s) from each schedule's interval or boundary-list specification, and builds and registers the resulting job and trigger definitions with the clustered scheduler at startup.
- **Startup reconciliation** — also at startup, before the scheduler begins running, removes any triggers left over from a prior configuration so nothing fires against a stale schedule.
- **Clustered scheduler layer** — fires triggers such that exactly one pod across the deployment handles any given firing, with job recovery enabled so a firing is never silently lost if its pod dies before completing — Quartz simply re-fires it on a live pod.
- **Admin control surface** — an authenticated, admin-only set of actions — Run now, Backfill, Pause, Resume, Status — addressed by (report type, frequency).
- **Window function** — a pure calculation that resolves a scheduled fire time to a (window start, window end) pair, exposed as a shared capability Assembly calls into. Every report frequency falls into one of three window shapes:
  - **Rolling** — a fixed look-back ending at the fire time (e.g. the last 30 minutes).
  - **Boundary** — the stretch since the previous checkpoint today, with midnight as an implicit first checkpoint.
  - **Calendar-day** — the entire previous calendar day, delivered the next morning.

  All window math happens in one configured business timezone (Europe/Stockholm) and is converted to absolute time for the outgoing request.

  Daylight-saving is handled explicitly. A fire time that falls in the skipped spring-forward hour shifts forward to the next real moment; a fire time in the repeated autumn hour fires only at its first occurrence, not the second. A Boundary window that merely spans a transition, with no boundary sitting inside the gap or overlap, is unaffected in count but not in length — the elapsed time it covers can differ by an hour on a transition day (the end-of-day window, for instance, is 23 hours on the spring transition and 25 on the autumn one). A Rolling frequency caught directly in the gap or overlap ends up with a window whose start and end resolve to the same instant — an empty report, not a wrong one. No Boundary checkpoint currently sits inside the 02:00–03:00 transition window, so the boundary-shift rules are not exercised in practice today, though they remain in place for whenever a boundary list changes to include one.

### Frequency catalogue

| Report type(s) | Frequency | Shape | Fires at | Days | Reporting window |
|---|---|---|---|---|---|
| CAMT052B, CAMT052BT | Every 30 min (`EVERY_30_MIN`) | Rolling | 00:30 → 21:00 | Mon–Fri | last 30 min |
| CAMT052B, CAMT052BT | Every hour (`EVERY_1_HOUR`) | Rolling | 01:00 → 21:00 | Mon–Fri | last 60 min |
| CAMT052B, CAMT052BT | Every 2 h (`EVERY_2_HOURS`) | Boundary | 03:00, 05:00 … 21:00 | Mon–Fri | since previous checkpoint; first run covers 00:00–03:00 |
| CAMT052B, CAMT052BT | Every 4 h (`EVERY_4_HOURS`) | Boundary | 05:00, 09:00, 13:00, 17:00, 21:00 | Mon–Fri | since previous checkpoint; first run covers 00:00–05:00 |
| CAMT054C | Once a day (`ONCE_PER_DAY`) | Boundary | 21:00 | Mon–Fri | 00:00–21:00, same day |
| CAMT054C | 4× a day (`FOUR_TIMES_PER_DAY`) | Boundary | 10:00, 13:00, 18:00, 21:00 | Mon–Fri | since previous checkpoint; first run covers 00:00–10:00 |
| CAMT054C | 8× a day (`EIGHT_TIMES_PER_DAY`) | Boundary | 03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00 | Mon–Fri | since previous checkpoint; first run covers 00:00–03:00 |
| CAMT053S, CAMT053E, CAMT054D | End of day (`END_OF_DAY`) | Calendar-day | 06:00 | Tue–Sat | the whole previous calendar day |
| any of the above | Never (`NEVER`) | — | not scheduled | — | — |

`NEVER` is the marker for configurations that are recipients of the inbound account-balance-push flow, triggered by an event rather than a timetable — no schedule ever picks a `NEVER` config up; the scheduled selection query structurally excludes it. Separately, 21:00 → 24:00 is never reported same-day by any Rolling or Boundary frequency, since their last checkpoint is 21:00.

### Interface to Assembly

- Scheduling invokes Assembly's scheduled-trigger entry point directly when a registered trigger fires, passing it that trigger's own configuration data (report type, frequency, scheduled fire time).
- Assembly calls Scheduling's window function to resolve a fire time into a (window start, window end) pair.
- A test-only override lets a schedule fire on a fast cadence instead of waiting for real clock time. This still produces a fresh, distinct request on every tick, even for End-of-day, because Assembly derives each scheduled request's identity from the scheduled fire time rather than the reporting window — a different fire time is a different identity, regardless of how often the window itself actually changes.

## 1.7 Workflow / Process Flow

### A. Startup

1. The deploy-time schedule configuration is read.
2. It is validated. Startup fails loudly if: a schedule's frequency does not parse to a known value; a Boundary schedule's boundary list is empty, not strictly ascending, or includes 00:00 explicitly; a schedule has neither an interval nor a boundary list (or, for Calendar-day, no fire-at time); the same (report type, frequency) appears in more than one entry; or an active report configuration exists whose (report type, frequency) has no matching schedule at all — a case that would otherwise silently never produce anything.
3. Any entry that lists several report types is expanded into one independent schedule per report type, and the underlying cron timing expression(s) are generated from each schedule's interval or boundary-list specification.
4. The resulting job and trigger definitions are built and registered with the clustered scheduler.
5. Any triggers left over from a prior configuration are cleaned up, before the scheduler begins running.
6. The clustered scheduler starts.

### B. Admin actions

- **Run now** — fires the schedule immediately, using the current moment as the scheduled fire time, and invokes Assembly's entry point the same way a normal firing would.
- **Backfill** — fires the schedule again for a specific missed slot, passing that slot's original scheduled time rather than the current moment, so the report covers the period it was always meant to. Available for any missed slot, sub-daily or end-of-day alike. Safe to reissue for a slot that already ran — invoking Assembly's entry point twice for the same slot is a no-op on Assembly's side (see the main solution document, Assembly). Backfill only ever re-runs a slot that a real schedule would have produced, using that slot's own window; producing a report for an arbitrary historical window or an arbitrary list of configuration ids is an on-demand request instead, not a Backfill.
- **Pause** — stops a schedule, addressed by report type and frequency, from firing going forward. Used to protect production during an environment issue, or once an operator has been informed of a database or message-queue problem.
- **Resume** — resumes a paused schedule; firing continues at its next natural time.
- **Status** — reports a schedule's paused/active state, plus its last and next fire time.

Misfire policy: a firing that is missed, whether from an outage or a deliberate pause, is not automatically caught up. The timetable simply resumes at its next natural time. Recovering a missed slot is always a deliberate action — Backfill, above.

Daylight-saving handling is covered under 1.6, Window function.
