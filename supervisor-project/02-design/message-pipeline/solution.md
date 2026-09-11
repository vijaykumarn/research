# Commander — Message Pipeline Solution

This is the reference for how Commander turns *"this report is due"* into a delivered
message — reliably, and without ever landing two different-looking copies at Executor, even
though the pods doing the work can be killed at any moment, mid-task, with no warning.
(Delivery itself is at-least-once, not exactly-once — §3 explains why that's still safe.)

> Visual: `pipeline-flow.drawio` (draw.io / diagrams.net) — the end-to-end flow, the outbox +
> delivery guarantee, recovery, and the `WorkItem` state machine.

> Building or reviewing the code? `solution_v08.md` has the exact table schemas, SQL, and the
> full reasoning behind each choice this document leaves out on purpose, to stay readable.

---

## 1. What the pipeline is responsible for

Commander's job is only this:

> decide which reports are due, for whom, for what time window → gather the account data →
> build a message → put it on the right queue.

Another application, **Executor**, picks those messages up and produces the actual reports.
Commander never renders anything itself.

It runs as **many identical copies at once** (pods), sharing one database and one message
queue. Any pod can be killed at any moment, mid-work. Everything else in this document exists
to make that survivable — without losing work, and without sending the same report twice.

---

## 2. The core idea: write the plan down before doing it

A naive service would fetch the list of reports to make, build each one, send it, and hope no
pod dies in between. If one does, there's no record of what was actually finished.

Commander instead **records its progress as database rows as it goes**. Nothing important ever
lives only in a pod's memory. Every step either updates a row or is safe to repeat — so if a
pod dies, another pod can read those rows and carry on from wherever the dead one stopped.

Two tables carry the weight:

- **A to-do list.** One row means "produce this one specific report message." All the to-do
  rows are written *before* any message is actually built.
- **An outbox tray.** A finished message is written here first — not sent straight to the
  queue. A separate helper (the **relay**) continuously takes messages out of the tray and puts
  them on the right queue.

**Why the tray, instead of just sending?** "Save to the database" and "send to the queue" are
two different systems — there's no way to do both as one all-or-nothing step. Send first, then
crash before recording it, and you don't know you sent it, so you send it again. Record it
first, then crash before sending, and it never goes out. The tray removes that gap: writing the
message to it *is* the moment it counts. Sending afterwards is just a follow-up step that can
always be safely retried — the worst case is explained next.

---

## 3. How duplicates are prevented

Every message has a **fingerprint** — an identity describing exactly what it represents: which
config, which report type, which account-or-payment-type slice, which time window, which
trigger, and which specific occurrence of that trigger produced it. That last piece is minted
fresh for on-demand and PHT (there's no caller-supplied id to reuse); for a scheduled run it's
derived from exactly which slot it was scheduled for, so the same slot always reproduces the
same identity while a genuinely different firing never does.

**The outbox tray refuses to hold two rows with the same fingerprint.** If the same message
somehow gets built twice — most often when a scheduled run and its own crash-recovery both
reach the same item — the first row goes in, the second is rejected, and the pod that lost sees
"already there, nothing to do" and moves on. One copy, ever. That's the real guarantee; there's
no separate lock involved.

**Scheduled and on-demand runs are deliberately independent, not deduplicated against each
other.** The fingerprint includes *which trigger*, so a scheduled message and an on-demand
request for the same config and window are two genuinely different fingerprints — both get
built, both get delivered, and Executor tells them apart by the trigger metadata carried in
each. That's intended: an on-demand request always produces a fresh message, even if a
scheduled run already covered the same window.

**PHT is kept separate by data, not by a lock.** The accounts that receive the external
balance-push flow have a configuration marked with a special frequency that the timetable-based
trigger never selects — so the scheduler simply never touches them, and PHT resolves its own
configuration directly by recipient.

**One honest caveat:** the hop from the tray to the queue can still deliver the *same* tray row
twice — the send can succeed, then the pod dies before marking it sent, and another pod resends
it. So every message carries its own fingerprint as fields inside it, and Executor is expected
to ignore a fingerprint it has already processed. Delivery to Executor is **at least once, not
exactly once** — the fingerprint is what makes that safe.

---

## 4. The three triggers, step by step

All three end up in the same core loop: *resolve the data → build the message → write it to
the outbox → mark the to-do item done.*

**Scheduled.** The scheduler fires (see `../scheduling/solution.md` for when and how often) —
clustered, so exactly one pod picks up each firing. That pod creates one tracking record for
this run, then works through the matching report configurations in pages of around 500 at a
time: fetch the next page, pull all the account and payment-type data for that whole page in
one batch (not one query per config), decide how many messages each config produces (§5), write
those as to-do rows, then for each one build the message, write it to the outbox, and mark it
done — before moving to the next page. Meanwhile the relay is continuously draining finished
messages from the outbox onto the report type's queue, in parallel with all of this.

**On-demand.** A message arrives on the on-demand queue carrying a list of config ids. The pod
that picks it up mints a fresh identity for this request — there's no id supplied by the
caller — creates a tracking record, and runs the same resolve → build → outbox → done loop for
just those configs. It then records the incoming message's id as handled and acknowledges the
queue. That minted identity is part of the fingerprint, which is why asking for the same report
twice on purpose produces two messages rather than being silently swallowed, and why an
on-demand message never collides with a scheduled one for the same window.

**PHT (the external balance push).** A fixed-width text message arrives carrying account
balances. The pod parses it, works out the recipient from an identifier in the message, and
looks up that recipient's configuration — the one the scheduler never touches. It mints an
acceptance id (same reason as on-demand), builds one message carrying those pushed balances,
writes it to the outbox, records the incoming message as handled, and acknowledges the queue.

---

## 5. What "one to-do item" means — the bundling rule

A single report configuration can turn into one message or many, depending on how it's set up:

| Configuration | Produces |
|---|---|
| **Bundled** | One message per payment type, covering all of that type's accounts |
| **Unbundled** | One message per account or alias |
| **No scope attached** | One "just the configuration" message |

The to-do rows are written to match this — one row per message that will actually be produced.
That's also why the to-do list can only be written *after* a configuration's data has been
resolved: there's no way to know the count before then.

---

## 6. When a pod dies

**A scheduled run.** While a run is active, its pod updates a heartbeat on the tracking record
every so often. A dedicated watchdog job — also clustered — watches for tracking records whose
heartbeat has gone stale, meaning the owning pod died. It claims that run (a compare-and-swap,
so only one restarting pod wins it) and resumes it in two parts: first it re-fetches and
rebuilds any to-do items that were still unfinished — safe to redo, because the outbox
fingerprint rule simply rejects anything already built before the crash — and then it keeps
paging forward from wherever the dead pod left off, exactly as normal processing would. If a
run keeps failing recovery past a set number of attempts, the watchdog marks it abandoned and
raises an alert for a person. (There's one narrow edge — a pod dying in the instant before it
even manages to create the tracking record — and that's covered by the scheduler's own
recovery, not this watchdog; see `../scheduling/solution.md` §8.)

**An on-demand or PHT message.** Much simpler — no watchdog needed. These arrived as queue
messages, so if a pod dies before finishing, the queue itself automatically redelivers the
message to another pod, which starts over. To avoid reprocessing a message that actually
finished just before the crash-and-redeliver, Commander records the id of every incoming
message it completes and skips any it's already seen.

---

## 7. How a to-do item ends

Every to-do item finishes in exactly one of these states:

| State | Meaning |
|---|---|
| **Published** | Its message is in the outbox. Done. |
| **Skipped (flag off)** | A feature flag said not to produce this one. Logged, done, not an error. |
| **Failed (poison)** | Tried several times, keeps failing — usually bad data. Terminal, and it **raises an alert**; the rest of the run carries on regardless. |
| **Obsolete** | During recovery, the account or scope this item was for turned out to have been genuinely removed. Logged, done, **not** an alert. |

**Obsolete only applies when the lookup succeeded and clearly showed the scope is gone.** A
lookup that merely failed or timed out is treated as an ordinary failure and retried — a
temporary database hiccup can never quietly retire real work.

---

## 8. A few reliability details worth knowing

- **A scheduled run and an on-demand request can target the same config and window on
  purpose**, and both are produced and delivered — they carry different fingerprints, so
  neither blocks or suppresses the other. This was a deliberate product decision, not an
  oversight: an on-demand request should never be silently swallowed just because the scheduler
  already covered that window.
- **Feature flags are checked per report before it's built or published.** Off means the item
  is marked skipped and the run carries on.
- **No ordering guarantee between messages.** The relay is sharded for throughput, so it makes
  no promise about the order messages land on the queue in. This is safe because every message
  is self-contained — it carries its own window and identity — and Executor is expected to
  process each one independently. (Confirming this as an explicit contract with the Executor
  team is still open, §10.)
- **The reporting window always reflects when a run was scheduled for, not when it actually
  ran.** It's computed once, from the trigger's scheduled time, and frozen on the tracking
  record — so a delayed or recovered run still produces the exact window it was meant to.
- **There's no lock between the three triggers.** Correctness rests entirely on the outbox
  fingerprint rule (§3) — nothing coordinates scheduled, on-demand, and PHT beyond that.
- **The chosen shape is deliberately simple:** one straightforward pipeline per to-do item
  (resolve → build → outbox → done), shared by all three triggers. A batch-processing framework
  was considered and not used — it restarts at a coarser granularity than a single item, adds
  its own bookkeeping, and doesn't fit event-driven triggers like on-demand and PHT well.

---

## 9. The tables, in one line each

| Table | In plain words |
|---|---|
| **Run** | One row per triggered run — a schedule firing, or an accepted on-demand / PHT message. Holds the heartbeat and how far it's got. |
| **WorkItem** | The to-do list: one row per message Commander intends to produce, plus its state and attempt count. |
| **Outbox** | The tray: finished messages waiting for the relay, one row per fingerprint, ever. |
| **ProcessedInboundMessage** | Ids of incoming queue messages already fully handled, so a redelivery doesn't cause a second run. |

---

## 10. What's still open

Not open design questions — three things to pin down before this is final:

- **How long to keep `ProcessedInboundMessage` ids** — read this off the actual redelivery /
  backout-queue configuration for the on-demand and PHT queues, rather than picking a number
  arbitrarily.
- **Confirm with the Executor team** that they dedupe incoming messages on the fingerprint, that
  they accept two semantically-equal messages distinguished only by trigger metadata (§8), and
  that they're fine processing messages with no ordering guarantee (§8).
- **Confirm or drop the on-demand safety check** for a mistakenly-supplied PHT-only config id —
  currently proposed as skip-and-log rather than failing the whole request, pending product
  sign-off on whether it's needed at all.
