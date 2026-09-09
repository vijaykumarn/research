# Feedback on `scheduling.md` — for the next revision pass

Reviewed against `goal.md`, `solutions_v08.md`, `how-it-works.md`, `faq.md`, `review-log.md`,
and `scheduling_goal.md`. Most of what a first pass would flag is already resolved in this
document (report-type-grouping vs. single-report-type `Run`, cron-per-trigger limits, the
exact-boundary-match fragility, the `END_OF_DAY` fast-TEST problem). What remains:

---

## 1. Blocking — `Run` schema is missing a column this design depends on

`scheduling.md` builds on `UNIQUE (report_type, frequency, scheduled_time)` on `Run` (referenced
in §1, §7, §9, §10, §16). **`solutions_v08.md`'s `Run` table has no `frequency` column** —
only `run_id, trigger_type, report_type, scheduled_time, execution_id, status, owner_pod,
started_at, heartbeat_at, last_config_id_processed, recovery_attempt_count`.

This is two separate problems, not one:

- **The uniqueness constraint can't be built as specified.** CAMT054C's three cadences
  (`ONCE_PER_DAY`, `FOUR_TIMES_PER_DAY`, `EIGHT_TIMES_PER_DAY`) all include a 21:00 boundary —
  `scheduling.md` §2 says as much itself. Without `frequency` in the key, three legitimate,
  differently-windowed runs for the same report type at the same `scheduled_time` on the same
  day collide on a `(report_type, scheduled_time)`-only constraint.
- **Recovery can't resume paging without it.** The scheduled selection predicate is
  `WHERE report_type = ? AND frequency = ? AND is_active = 1`. One report type can have
  several frequencies (CAMT054C has three). If a pod dies mid-run, the sweeper resuming that
  `Run` past `last_config_id_processed` has no way to know *which* frequency's config set to
  keep paging through unless `frequency` is persisted on the `Run` row.
- **Related:** `Run` also has no `window_start`/`window_end`, only `scheduled_time`. The
  Foundation section of `solutions_v08.md` says "window ... stored on `Run`," but the DDL
  doesn't store one — presumably it's meant to be recomputed on demand via `scheduling.md`'s
  window function. That function's shape (rolling / boundary / calendar-day) depends on
  `frequency` too — another independent reason the column needs to exist, on top of the
  uniqueness argument above.

**Fix:** add `frequency VARCHAR(20) NOT NULL` to `Run` in `solutions_v08.md`, and make the
constraint explicit in the DDL: `CONSTRAINT UQ_Run_Slot UNIQUE (report_type, frequency,
scheduled_time)`. This is a `solutions_v08.md` change, but it blocks `scheduling.md` from being
internally consistent with the schema it relies on — fix both together.

---

## 2. Needs reconciling — `EVERY_2_HOURS`/`EVERY_4_HOURS` window shape

`scheduling.md` §12 lists this as an **open business decision** between `ROLLING` and
`BOUNDARY` shape, citing "current production does not cover 00:00 → first fire for these two."

But `scheduling_goal.md`'s cadence table already states this as a **fixed requirement**:
`EVERY_2_HOURS`'s window is "previous firing's time to this firing's time; **first window
00:00 → 03:00**" — boundary-shaped, not open.

These two documents now assert different things as settled fact for the same requirement. This
may be a legitimate correction (production genuinely doesn't anchor to midnight today, and that
fact only surfaced during this design pass) — but as written there's no explanation of where
the new claim comes from or why it overrides the brief. Resolve one of two ways:

- If it's a genuine correction: update `scheduling_goal.md`'s cadence table to match, so it
  stops reading as a settled requirement elsewhere, and note in `scheduling.md` §12 that this
  supersedes the brief and why.
- If it's still genuinely undecided: soften `scheduling_goal.md`'s phrasing too, so the two
  documents don't contradict each other about whether this was ever settled.

Either is fine — just don't leave one document calling it fixed and the other calling it open
with no cross-reference explaining the discrepancy.

---

## 3. Minor — `Run` creation hitting the unique constraint needs a named-success-path statement

`scheduling.md` §7 says a repeated firing for the same slot makes `Run` creation "fail fast" on
the unique constraint, but doesn't say how the job should handle that outcome.
`solutions_v08.md` was explicit that an `Outbox` unique-violation is a *named success path*
("already exists → select the existing row and mark `PUBLISHED`, not an error"). Say the same
thing here: does the job catch the violation and exit cleanly, or does it let Quartz see an
exception (which could trigger unwanted misfire/alerting behavior)? Given the pattern
established elsewhere, the answer is presumably "catch and exit cleanly," but it isn't stated.

---

## Confirmed correct — no action needed

For completeness, so the fixing pass doesn't waste time re-litigating these: the rolling/boundary
equivalence for `EVERY_30_MIN`/`EVERY_1_HOUR`'s first-of-day window, the derived-`execution_id`
trick that solves `END_OF_DAY` fast-cadence testing without a tick-based redesign, the
`requestRecovery(false)` reasoning (pipeline sweeper owns recovery, not Quartz), the
multi-`TriggerKey`-per-logical-schedule handling in pause/resume (§14), the off-grid
skip guard in the window function (§4), and the physical-trigger-count arithmetic in §5 all
check out against `goal.md`, `faq.md`, and `solutions_v08.md`.
