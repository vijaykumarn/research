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

## 1.6 Logical View - Architecture Scope

- **Trigger loader / generator** — reads the deploy-time schedule configuration, expands any entry that lists several report types into one independent schedule per report type, generates the underlying cron timing expression(s) from each schedule's interval or boundary-list specification, and builds and registers the resulting job and trigger definitions with the clustered scheduler at startup.
- **Startup reconciliation** — also at startup, before the scheduler begins running, removes any triggers left over from a prior configuration so nothing fires against a stale schedule.
- **Window function** — a pure calculation that resolves a scheduled fire time to a (window start, window end) pair, exposed as a shared capability Assembly calls into. Every report frequency falls into one of three window shapes:
  - **Rolling** — a fixed look-back ending at the fire time (e.g. the last 30 minutes).
  - **Boundary** — the stretch since the previous checkpoint today, with midnight as an implicit first checkpoint.
  - **Calendar-day** — the entire previous calendar day, delivered the next morning.

  All window math happens in one configured business timezone (Europe/Stockholm) and is converted to absolute time for the outgoing request. Daylight-saving is handled explicitly: a fire time in the skipped spring hour shifts forward to the next real moment; a fire time in the repeated autumn hour fires only at its first occurrence.
- **Clustered scheduler layer** — fires triggers such that exactly one pod across the deployment handles any given firing, with job recovery enabled so a firing is never silently lost if its pod dies before completing — Quartz simply re-fires it on a live pod.
- **Admin control surface** — an authenticated, admin-only set of actions — Run now, Backfill, Pause, Resume, Status — addressed by (report type, frequency).

### Interface to Assembly

- Scheduling invokes Assembly's scheduled-trigger entry point directly when a registered trigger fires, passing it that trigger's own configuration data (report type, frequency, scheduled fire time).
- Assembly calls Scheduling's window function to resolve a fire time into a (window start, window end) pair.

## 1.7 Workflow / Process Flow

### A. Startup

1. The deploy-time schedule configuration is read.
2. It is validated.
3. Any entry that lists several report types is expanded into one independent schedule per report type, and the underlying cron timing expression(s) are generated from each schedule's interval or boundary-list specification.
4. The resulting job and trigger definitions are built and registered with the clustered scheduler.
5. Any triggers left over from a prior configuration are cleaned up, before the scheduler begins running.
6. The clustered scheduler starts.

### B. Admin actions

- **Run now** — fires the schedule immediately, using the current moment as the scheduled fire time, and invokes Assembly's entry point the same way a normal firing would.
- **Backfill** — fires the schedule again for a specific missed slot, passing that slot's original scheduled time rather than the current moment, so the report covers the period it was always meant to. Available for any missed slot, sub-daily or end-of-day alike. Safe to reissue for a slot that already ran — invoking Assembly's entry point twice for the same slot is a no-op on Assembly's side (see the main solution document, Assembly).
- **Pause** — stops a schedule, addressed by report type and frequency, from firing going forward. Used to protect production during an environment issue, or once an operator has been informed of a database or message-queue problem.
- **Resume** — resumes a paused schedule; firing continues at its next natural time.
- **Status** — reports a schedule's paused/active state, plus its last and next fire time.

Misfire policy: a firing that is missed, whether from an outage or a deliberate pause, is not automatically caught up. The timetable simply resumes at its next natural time. Recovering a missed slot is always a deliberate action — Backfill, above.

Daylight saving: a fire time that falls in the skipped spring-forward stretch shifts forward to the next real moment; one that falls in the repeated autumn stretch fires once, at its first occurrence.
