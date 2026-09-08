# Commander — How It Works, In Plain Terms

This explains what the application actually *does*, step by step, and why it's built that way.
No schema detail — just the mechanism. Reflects the **v08** design. Scheduling (which report
type runs how often, and the reporting-window rules) is a separate concern — see
`scheduling.md`.

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

## 3. How duplicates are prevented

Every message has a **logical identity** — a fingerprint of what it represents:

> which config · which report type · which account-or-payment-type slice · which time window
> · which trigger · (for on-demand/PHT) which request

The **Outbox table refuses to hold two rows with the same fingerprint.** So if the same
message gets built twice — most commonly when a scheduled run and its own crash-recovery both
reach the same item, or a slow pod is still alive when recovery starts — the first row goes in
and the second insert bounces. The second pod sees "already there, nothing to do" and moves
on. One copy, ever. That database rule is the real guarantee; there is no separate lock.

**Scheduled and on-demand are deliberately independent, not deduplicated against each other.**
The fingerprint includes *which trigger* and, for on-demand, *which request*. So a scheduled
"CAMT052B, config X, window W" and an on-demand request for the same config and window have
**different** fingerprints — both are built, both are published, and Executor tells them apart
by the trigger metadata. That is intended: an on-demand request always produces a fresh
message, even if the scheduler already made one for the same window.

**PHT is kept separate by data, not by a lock.** PHT recipients have a `report_config` row
whose frequency is set to `NEVER`, and the scheduled path only ever selects rows matching a
real report-type + frequency + active filter — so it never touches them. The PHT flow looks
that config up by recipient and report type directly, frequency notwithstanding.

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
   - fetch the next 500 `ReportConfig`s matching this report type + this frequency + active
   - one set of bulk queries pulls all the account / alias / payment-type data for those 500
     at once (not 500 separate trips)
   - for each config, the **bundling rule** (section 5) says how many messages it produces —
     write those as **WorkItem** rows
   - for each WorkItem: build the message → write it to the **Outbox** → mark the WorkItem done
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
same report twice on purpose produces two messages instead of being silently swallowed, and
why an on-demand message never collides with a scheduled one.

### PHT (external balance push)

1. A fixed-width text message lands on `CAMT.PHT.QUEUE` carrying account balances.
2. A pod parses it, works out the recipient from the engagement identifier in the message, and
   looks up that recipient's `CAMT052B` config — which is marked `frequency = NEVER`, so the
   scheduler never produces for it. It **mints an acceptance id** (same reason as on-demand).
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

## 6. What happens when a pod dies

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

## 7. How a WorkItem ends

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

## 8. The tables, in one line each

| Table | In plain words |
|---|---|
| **Run** | One row per triggered run — a schedule firing, or an accepted on-demand / PHT message. Holds the heartbeat and the "finished up to config X" mark. |
| **WorkItem** | The to-do list: one row per report message Commander intends to produce, plus its state and attempt count. |
| **Outbox** | The outbox tray: finished messages waiting for the relay to put them on MQ. Enforces one-row-per-fingerprint. |
| **ProcessedInboundMessage** | Ids of incoming queue messages already fully handled, so an MQ redelivery doesn't cause a second run. |

---

## 9. Why this shape achieves the goals

| Goal (from the brief) | How it's met |
|---|---|
| Don't lose work when a pod is killed mid-run | Progress is all database rows, never pod memory. A watchdog resumes stale scheduled runs from where they stopped; MQ redelivers on-demand / PHT. |
| Never publish a duplicate report message | The Outbox refuses a second row with the same fingerprint — only one copy can ever enter the tray. |
| …and if MQ still delivers a repeat | Each message carries its fingerprint; Executor ignores fingerprints it has already processed. |
| Only one pod runs a given scheduled trigger | Quartz clustered mode, for free. |
| A scheduled run and an on-demand request for the same config/window | Both are produced and delivered, tagged by trigger type + execution id — they have different fingerprints, so neither blocks or suppresses the other. PHT recipients are held separate by a `frequency = NEVER` config the scheduler never selects. |
| Handle 1,000–10,000 configs per run efficiently | Work in pages of ~500; one set of bulk queries per page, not per config. |
| One bad record shouldn't stall the whole run | Per-item attempt counter → `FAILED_POISON` + alert; the run keeps going. |
| The report window must reflect the scheduled time, not when the pod happened to run | The window is computed from the trigger's scheduled time and stored on the Run row, so a delayed or resumed run still produces the window it was meant to. |

---

## 10. Chosen design and what's still open

**Chosen:** "Option A" — one straightforward in-process pipeline per WorkItem
(resolve → assemble → Outbox → mark done), shared by all three triggers. Spring Batch was
considered and not adopted (it restarts at the step level, adds its own bookkeeping tables,
and doesn't fit the event-driven on-demand / PHT triggers).

**Still to pin down (numbers and a diagram, not design):**
- how long to keep `ProcessedInboundMessage` ids (from the MQ redelivery / backout settings)
- confirming with the Executor team that they will dedupe on the message fingerprint, and that
  they accept two semantically-equal messages distinguished only by trigger metadata
- the scheduling design itself — covered separately in `scheduling.md`
