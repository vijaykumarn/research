# Commander Redesign — FAQ

Plain-language explanations of points raised in `goal.md`. Kept here so the reasoning isn't lost.

---

## Q1. "All three triggers publish to MQ, so the duplicate rule applies to on-demand and PHT too" — what does this mean?

There are **two different places** a failure can hit:

- **Reading the trigger in** — pulling the on-demand or PHT message off its inbound queue.
- **Writing the report out** — putting the finished `ReportMessage` on the report-type queue.

Queue redelivery only protects the *first* one. If the message broker doesn't get an
acknowledgement, it hands the inbound message to another pod — so you won't lose the request.

But this sequence is still possible:

1. A pod reads a PHT message.
2. It builds the report and **successfully sends it** to `CAMT.052B.QUEUE`.
3. The pod crashes *before* it acknowledges the inbound PHT message.
4. The broker thinks the PHT message was never handled, so it redelivers it.
5. Another pod picks it up, builds the same report, and sends it **again**.

Executor now has two copies of the same report. Redelivery fixed the crash but caused a
double-send.

So "on-demand and PHT can just rely on queue retry" is only half true — it makes the *input*
safe but leaves the *output* able to duplicate. Whatever mechanism stops the scheduled path
from double-publishing has to wrap the publish step for on-demand and PHT as well.

*Analogy: a cashier charges your card, then the register crashes before printing the receipt.
On restart it doesn't know the charge went through, so it charges you again.*

---

## Q2. "Concurrent triggers can overlap" — what does this mean?

Commander has three independent front doors — the scheduler, the on-demand queue, the PHT
queue — and nothing coordinates them. Plus there are many pods running at once.

So this can happen:

- **10:00** — the scheduled `CAMT052B` job fires and starts working through 3,000 accounts.
- **10:01** — someone submits an on-demand request: "regenerate `CAMT052B` for account X."
  Account X happens to be one of the 3,000 the scheduled run hasn't reached yet.
- Now **two different pods** are building a `CAMT052B` report for account X, for roughly the
  same time window, at the same time.

Bad outcomes:

- Executor gets two `CAMT052B` messages for account X for the same window — which is the real
  one?
- The two runs both touch the same tracking rows in the database and one overwrites the other.
- Duplicate detection treats them as "the same report" and silently drops one — possibly the
  one the user actually asked for.

The design needs a way to say: *while a report for this scope + window is being produced,
nobody else starts producing the same one.* A lock or a claim on that scope that all three
trigger paths honour.

*Analogy: two people editing the same spreadsheet cell at the same moment with no locking —
last save wins, the other edit is lost, or you end up with two conflicting versions.*

---

## Q3. "Data resolution at volume — batched per page, not per config" — what does this mean?

To build one report, Commander walks a tree of data for each `ReportConfig`:

```
ReportConfig
  → agreement scope(s)
    → payment types
      → accounts / aliases
```

Each level is a separate SQL Server table.

**The slow way (per config):** loop over configs one at a time; for each one, run a few
queries — fetch its scopes, then its payment types, then its accounts, then its aliases.
That's ~4–5 database round-trips *per config*. A scheduled run with 5,000 configs →
**~25,000 round-trips** for one run. Slow, and it hammers the database.

**The fast way (per page):** process configs in pages — say 500 at a time. For a page, run
**one** query that fetches the scopes for all 500 configs at once, one for all their payment
types, one for all their accounts, one for all their aliases — then assemble the 500 trees in
memory. That's ~4–5 queries *per page*, so **~50 queries** for the whole run instead of
25,000.

The point for the architecture options: assume the data-access layer reads in batches keyed
by a page of config IDs. Per-config reads won't survive this system's volumes.

*Analogy: grocery shopping. Per-config = drive to the store, buy one item, drive home, repeat
5,000 times. Per-page = one trip, fill the cart.*

---

## Q4. Is there a caller-supplied request / correlation ID on the on-demand message that dedup can key off?

**No.** The current on-demand message contract carries only: report type, report version, a
list of config identifiers, and the reporting period. The design should not assume a
business-level request ID exists.

Two separate problems hide in this question — keep them apart:

**(a) Broker redelivery** — a pod crashes after producing output but before acknowledging the
inbound on-demand message, so the broker hands the same message to another pod. This does
*not* need a business request ID. JMS already provides what's needed: every inbound message
has a `JMSMessageID`, and a redelivered one arrives with `JMSRedelivered = true` and the
**same** `JMSMessageID`. Commander should persist the `JMSMessageID` of each on-demand
message it has fully processed and skip any it has already seen. A genuine resubmit is a
fresh send → new `JMSMessageID` → processed normally. This cleanly separates "the broker gave
me this again" from "the caller asked again." (Needs a small processed-inbound-IDs store with
a retention window — can share the outbound dedup ledger.)

**(b) The on-demand run's identity inside the outbound dedup key** — the outbound key (the
thing that stops a duplicate reaching Executor) must not make an on-demand-produced message
collide with a scheduled-produced message for the same config + window, or a legitimate
on-demand regeneration gets suppressed. So mint a **Commander-side on-demand execution ID**
(a UUID) when the message is accepted off the queue, persist it on the on-demand run record,
and fold it into the dedup identity for messages that run produces. Scheduled-path messages
use a stable content-only key; on-demand-path messages use content key + execution ID.

**Optional improvement:** ask the upstream caller to add an explicit `requestId` to the
contract. It gives end-to-end traceability and removes the reliance on JMS redelivery
semantics for (a). Nice to have, not required.

---

## Q5. What granularity should the cross-trigger lock be — whole `ReportConfig`, or the message identity?

First, a correction that shrinks the question: a `ReportConfig` is unique per
(recipient, report type) — each config is for exactly **one** report type. So "an on-demand
request for report type A while a scheduled run for report type B is mid-flight on the same
config" can't happen; locking a config only ever blocks other work for that same report type.

**Recommended granularity: `(config identifier, reporting window)`** — not the whole config,
not the full message identity.

- **Not the whole config:** the only extra thing full-config locking buys is blocking two
  runs for *different* time windows of the same config — which is exactly what you don't want
  to block (an on-demand regeneration of yesterday's window shouldn't wait on a scheduled run
  doing today's).
- **Not finer (per payment type / account / page):** within one config + window, only one
  trigger should be producing at a time anyway. The sub-config fan-out needs no cross-trigger
  arbitration, and keying on it multiplies lock rows and forces both sides to normalize the
  fan-out identity identically.
- Acquire the lock **per config as the scheduled run reaches it**, not one lock for the whole
  run — otherwise a multi-thousand-config run blocks on-demand for any config in its scope
  for the run's entire duration.
- Release once that config's message(s) for that window are durably recorded.

**Framing that matters: the lock is an optimization, not the correctness guarantee.** The
real backstop against a duplicate reaching Executor is a `UNIQUE` constraint on the outbound
message's logical identity (in the outbox / sent-ledger). If two triggers race the same
`(config, window)` despite the lock, both build the message, both try to record it, and the
unique constraint rejects the loser — wasted CPU, no duplicate published. That lets the lock
be a short advisory claim rather than a correctness-critical distributed lock.

**On contention:** when an on-demand or PHT message can't take the lock because a scheduled
run holds that `(config, window)`, NACK it for redelivery with a short delay and bounded
retries — it comes back naturally after the scheduled run releases that config. "Processed
promptly, no fixed SLA" makes brief waiting acceptable.

> **Superseded in v08.** The `ScopeClaim` lock described here was removed — see
> `solutions_v08.md`. Scheduled and on-demand messages already have distinct logical
> identities, so both publish for the same `(config, window)` by design; `UQ_Outbox_Identity`
> alone covers the only real double-publish case (a scheduled run vs. its own recovery).

---

## Q6. What's the difference between `DAILY` and `ONE_TIME_PER_DAY` — why not merge them into one?

They are the **same cadence** — once per day. The only real difference is the **reporting-window
rule**:

| | `DAILY` (CAMT053S, CAMT053E, CAMT054D) | `ONE_TIME_PER_DAY` (CAMT054C) |
|---|---|---|
| Fires | 06:00, Tue–Sat | 21:00, Mon–Fri |
| Window | the **whole previous calendar day** (00:00–24:00 of yesterday) | **midnight → the fire time** of the **same** day (00:00–21:00) |
| Window model | `PREVIOUS_CALENDAR_DAY` | `BOUNDARY` (list `[00:00, 21:00]`) |

Fire time and days are just config. The substantive split is "yesterday, complete" vs. "today
so far, partial."

**You *can* technically merge them** — the trigger is keyed by `(report_type, frequency)`, so
`(CAMT053S, ONCE_PER_DAY)` and `(CAMT054C, ONCE_PER_DAY)` would already be different triggers
with their own cron, days, and window model, and no single report type needs both behaviours.

**But merging is not a good idea:**

1. **"Window model" is not an independent axis.** Every frequency in the catalogue maps to
   exactly one window rule (all boundary-model except `DAILY`). Today that mapping is derivable
   from the frequency. Merging `DAILY` and `ONE_TIME_PER_DAY` *creates* a genuine, settable-wrong
   `window-model` config field where there wasn't one — relocating complexity, not removing it.
2. **They are different report concepts.** `DAILY` is an end-of-day report on a *completed*
   day (statements, full-day notifications), run the next morning. `ONE_TIME_PER_DAY` is the
   **N = 1 member of CAMT054C's intraday family** — same report type, same partial-same-day
   semantics, same boundary model as its `FOUR_`/`EIGHT_TIMES_PER_DAY` siblings. It belongs
   *with* them; `DAILY` belongs with nothing else.
3. **`frequency` is a `ReportConfig` column people read.** Keep it self-describing. After a
   merge, the same `frequency` value would mean structurally different reporting depending on
   report type.
4. **It composes badly.** A future third once-daily window (previous *business* day, trailing
   24 h, …) is a new named value in the current design, versus an ever-growing `window-model`
   enum that every once-daily trigger must set correctly.
5. **Weaker misconfiguration guard.** `(CAMT053S, ONE_TIME_PER_DAY)` today has no matching
   trigger → caught at startup. Merged, it's valid and correctness rests entirely on a
   separate field being right.

**Cost of keeping them separate:** one extra enum constant. No runtime cost, no schema cost
(`frequency` is in no dedup key), no coupling.

**What was worth fixing is the naming** — `DAILY` and `ONE_TIME_PER_DAY` *sounded* like
synonyms. Resolved (applied in `scheduling.md`):

- `DAILY` → **`END_OF_DAY`** — CAMT053S / 053E / 054D; reports the completed previous day.
- `ONE_TIME_PER_DAY` → **`ONCE_PER_DAY`** — CAMT054C; the 1× member of the
  `ONCE_ / FOUR_ / EIGHT_TIMES_PER_DAY` family, partial same-day window.

Two distinct concepts, two names, no apparent overlap.
