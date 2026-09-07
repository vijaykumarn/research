# Commander (v07) — How It Works, In Plain Terms

This explains what the application actually *does*, step by step, and why it's built that way.
No schema detail — just the mechanism.

---

## 1. What the app is for

Commander produces **report messages** and drops them on IBM MQ queues. Another app,
**Executor**, picks them up and makes the real reports. Commander's job is only:

> decide which reports are due, for whom, for what time window → gather the account data →
> build a JSON message → put it on the right queue.

It runs as **many identical copies (pods)** at once, sharing one database and one MQ. Any pod
can be killed at any moment — mid-work, no warning. The whole design exists to make that
survivable without losing work or sending anything twice.

---

## 2. The one big idea: write down the plan before doing it

A naive service would: *get the list of reports to make → make each one → send it → hope no
pod dies in between.* If a pod dies halfway, you've lost track of what was done.

Commander instead **records everything as rows in the database as it goes**. Nothing important
lives only in a pod's memory. Every step either updates a row or is safe to repeat. So if a
pod dies, another pod can read the rows and carry on from where the dead one stopped.

Two tables carry the weight:

- **WorkItem** — a to-do list. One row = "produce this one specific report message." All the
  to-do rows are written *before* the messages are built.
- **Outbox** — an outbox tray. A finished message is written *here first*, not sent straight to
  MQ. A separate helper (the **relay**) takes messages out of the tray and puts them on the
  queue.

### Why the outbox tray instead of sending directly?

"Save to the database" and "send to MQ" are two separate systems. You cannot do both as a
single all-or-nothing step. So:

- Send to MQ, then crash before recording it → you don't know you sent it → you send it again.
- Record it, then crash before sending → it never goes out.

The outbox removes the gap: **writing the message to the Outbox table IS the moment it
"counts."** Sending is just a follow-up that can be retried safely. The worst case is the
relay sends the same tray row twice — handled in the next section.

---

## 3. How duplicates are actually prevented

Every message has a **logical identity** — a fingerprint of what it represents:

> which config · which report type · which account-or-payment-type slice · which time window
> · (for on-demand/PHT) which request

The **Outbox table refuses to hold two rows with the same fingerprint.** So if two pods ever
try to produce the same report at the same time, the first one's row goes in and the second
one's insert bounces — the second pod sees "already there, nothing to do" and moves on. One
copy, ever.

That database rule is the real guarantee. Everything else (the "claim" in section 6) is just
to avoid wasting effort.

**One honest caveat:** the relay → MQ hop can still deliver the *same tray row* twice (send
succeeds, pod dies before marking it "sent", another relay resends it). So the message carries
its fingerprint as fields inside it, and **Executor is asked to ignore a fingerprint it has
already seen.** Delivery to Executor is "at least once", not "exactly once" — the fingerprint
is what makes that safe.

---

## 4. The three triggers, step by step

All three end up in the same core loop: *resolve data → build message → write to Outbox → mark
done.*

### Scheduled

1. Quartz (the scheduler) fires, e.g. "CAMT052B, every 30 minutes". Quartz runs in **clustered
   mode**, which by itself guarantees **only one pod** picks up that firing.
2. That pod creates a **Run** row (one run = this firing).
3. It works in **pages of ~500 configs** at a time:
   - fetch the next 500 active `ReportConfig`s for this report type
   - one set of bulk queries pulls all the account / alias / payment-type data for those 500
     at once (not 500 separate trips)
   - for each config, the **bundling rule** (section 5) says how many messages it produces —
     write those as **WorkItem** rows
   - for each WorkItem: claim the config (section 6) → build the message → write it to the
     **Outbox** → mark the WorkItem done → release the claim
   - record "I've finished configs up to id X" on the Run row
   - next page
4. Meanwhile the **relay** is continuously taking finished messages out of the Outbox and
   putting them on `CAMT.052B.QUEUE`.

### On-demand

1. A message lands on `CAMT.ONDEMAND.QUEUE` carrying a list of config ids.
2. A pod picks it up and **mints a unique execution id** for this request (there's no id
   supplied by the caller).
3. It creates a Run, then runs the same *resolve → build → Outbox → mark done* loop for those
   configs.
4. It records the incoming queue message's id as "processed" and acknowledges the queue.

The execution id is part of the fingerprint for these messages — that's why asking for the
same report twice on purpose produces two messages instead of being silently swallowed.

### PHT (external balance push)

1. A fixed-width text message lands on `CAMT.PHT.QUEUE` carrying account balances.
2. A pod parses it, works out the recipient from the engagement identifier in the message, and
   **mints an acceptance id** (same reason as on-demand).
3. It builds one `CAMT052B` message that carries those pushed balances, writes it to the
   Outbox, records the incoming message id, and acknowledges the queue.

---

## 5. What "one WorkItem" means — the bundling rule

A single config can turn into one message or many:

| Config | Produces |
|---|---|
| **Bundled** | One message per payment type, covering all that payment type's accounts |
| **Unbundled** | One message per account / alias |
| **No scope attached** | One "just the config" message |

The WorkItem rows are created to match — one row per message that will be produced. That's
why the to-do list can only be written *after* the config's data is resolved (you can't know
the count before then).

---

## 6. Keeping two triggers off the same config at once

The scheduled run, an on-demand request, and a PHT push can all want the same config at the
same moment, on different pods. To avoid both building the same thing:

- Before working a config for a window, a pod writes a short-lived **claim** row for
  `(config, window)`.
- Another pod that wants the same `(config, window)` sees the claim and backs off (on-demand /
  PHT put their message back on the queue to retry shortly; the scheduled run lets its own
  recovery pass catch it later).
- The claim **auto-expires** after a set time, so a pod dying while holding one doesn't block
  that config forever.

The claim is only an efficiency measure. Even if it fails and two pods both build the message,
the Outbox fingerprint rule (section 3) still lets only one copy through.

---

## 7. What happens when a pod dies

### A scheduled run

- While a run is active, its pod updates a **heartbeat** timestamp on the Run row every so
  often.
- A dedicated **watchdog job** (also clustered) looks for Run rows whose heartbeat has gone
  stale — that means the pod died.
- It claims that run (a compare-and-swap so only one restarting pod wins it) and resumes it in
  two parts:
  1. **Re-do the unfinished WorkItems.** It re-fetches their data in bulk (per page, same as
     normal) and rebuilds them. Re-running a WorkItem is safe because of the Outbox fingerprint
     rule — if it was already done pre-crash, the insert just bounces and the item is marked
     done.
  2. **Keep paging forward** from the "finished up to config id X" mark, creating and
     processing the configs the dead pod never reached.
- If a run keeps failing recovery past a set number of attempts, the watchdog marks it
  **abandoned** and raises an alert for a human.

### An on-demand or PHT message

Much simpler — no watchdog needed. These arrived as queue messages. If the pod dies before
finishing, **MQ automatically redelivers** the message to another pod, which starts over.
To avoid re-processing a message that was actually completed just before the crash-and-redeliver,
Commander records the id of every incoming message it finishes and skips any it has seen
before.

---

## 8. How a WorkItem ends

Every WorkItem finishes in exactly one of these states:

| State | Meaning |
|---|---|
| **PUBLISHED** | Its message is in the Outbox. Done. |
| **SKIPPED_FLAG_OFF** | A feature flag said "don't produce this one." Logged, done, not an error. |
| **FAILED_POISON** | Tried several times, keeps failing (usually bad data). Terminal, **raises an alert** for a person. The rest of the run carries on. |
| **OBSOLETE** | During recovery, the account / scope this item was for turned out to have been genuinely removed. Logged, done, **not** an alert. |

`OBSOLETE` only applies when the data lookup *succeeded* and clearly showed the scope is gone —
a lookup that merely failed or timed out is treated as a normal failure and retried, so a
temporary database hiccup can never quietly retire real work.

---

## 9. The tables, in one line each

| Table | In plain words |
|---|---|
| **Run** | One row per triggered run — a schedule firing, or an accepted on-demand / PHT message. Holds the heartbeat and the "finished up to config X" mark. |
| **WorkItem** | The to-do list: one row per report message Commander intends to produce, plus its state and attempt count. |
| **Outbox** | The outbox tray: finished messages waiting for the relay to put them on MQ. Enforces one-row-per-fingerprint. |
| **ScopeClaim** | Short-lived "I'm working on this `(config, window)` right now" markers. Auto-expiring. |
| **ProcessedInboundMessage** | Ids of incoming queue messages already fully handled, so an MQ redelivery doesn't cause a second run. |

---

## 10. Why this shape achieves the goals

| Goal (from the brief) | How it's met |
|---|---|
| Don't lose work when a pod is killed mid-run | Progress is all database rows, never pod memory. A watchdog resumes stale scheduled runs from where they stopped; MQ redelivers on-demand / PHT. |
| Never publish a duplicate report message | The Outbox refuses a second row with the same fingerprint — only one copy can ever enter the tray. |
| …and if MQ still delivers a repeat | Each message carries its fingerprint; Executor ignores fingerprints it has already processed. |
| Only one pod runs a given scheduled trigger | Quartz clustered mode, for free. |
| Two triggers shouldn't build the same config at once | Short-lived `(config, window)` claim rows; the Outbox rule is the real backstop if a claim is missed. |
| Handle 1,000–10,000 configs per run efficiently | Work in pages of ~500; one set of bulk queries per page, not per config. |
| One bad record shouldn't stall the whole run | Per-item attempt counter → `FAILED_POISON` + alert; the run keeps going. |
| The report window must reflect the scheduled time, not when the pod happened to run | The window is computed from the trigger's scheduled time and stored on the Run row, so a delayed or resumed run still produces the window it was meant to. |

---

## 11. Chosen design and what's still open

**Chosen:** "Option A" — one straightforward in-process pipeline per WorkItem
(resolve → assemble → Outbox → mark done), shared by all three triggers. Spring Batch was
considered and not adopted (it restarts at the step level, adds its own bookkeeping tables,
and doesn't fit the event-driven on-demand / PHT triggers).

**Still to pin down (numbers and a diagram, not design):**
- how long a `ScopeClaim` should live before expiring (from a load test)
- how long to keep `ProcessedInboundMessage` ids (from the MQ redelivery / backout settings)
- a sequence diagram of the "scheduled run and on-demand request hit the same config at once"
  case
- confirming with the Executor team that they will dedupe on the message fingerprint
