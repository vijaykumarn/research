# Commander — Message Pipeline Solution

This is the reference for how Commander builds and delivers report messages — reliably, even
when the pods doing the work die mid-task.

> Visual: `pipeline-flow.drawio` (draw.io / diagrams.net) — the end-to-end flow, the outbox +
> delivery guarantee, recovery, and the `WorkItem` state machine.

> Building or reviewing the code? `solution_v08.md` has the exact table schemas, SQL, and the
> full reasoning behind each choice this document leaves out on purpose, to stay readable —
> `schema.dbml` has the same four tables as a diagrammable DB model.

---

## 1. What the pipeline is responsible for

Commander's job is only this:

> decide which reports are due, for whom, for what time window → gather the payment-type data
> (accounts or aliases) → build a message → put it on the right queue.

Another application, **Executor**, picks those messages up and produces the actual reports.
Commander's output *is* that message — an instruction for Executor to act on, describing what
to produce and for whom. Commander never renders anything itself; its responsibility ends the
moment that instruction is on the queue.

Commander runs as **many identical copies at once** (pods), sharing one database and one
message queue, and any one of those pods can be killed at any moment — mid-task, with no
warning and no graceful shutdown.

---

## 2. The core idea: write the plan down before doing it

A naive service would fetch the list of reports to make, build each one, and send it — without
taking care of reliability. If a pod dies partway through, there's no record of what was
actually finished.

Commander instead **records its progress as database rows as it goes**. Nothing important ever
lives only in a pod's memory. Every step either updates a row or is safe to repeat — so if a
pod dies, another pod can read those rows and carry on from wherever the dead one stopped.

Two tables carry the weight:

- **A to-do list.** One row means "produce this one specific report message." All the to-do
  rows are written *before* any message is actually built.
- **An outbox tray.** A finished message is written here first — not sent straight to the
  queue. A separate helper (the **relay**) continuously takes messages out of the tray and puts
  them on the right queue. Every pod runs its own relay loop at once — unlike a scheduled
  firing, which wants exactly one pod, the relay wants all of them draining in parallel, for
  throughput — and a claim on each row is what stops two pods sending the same one twice.

**Why the tray, instead of just sending straight to the queue?** "Save to the database" and
"send to the queue" are two different systems, so there's no way to do both together as one
all-or-nothing step — one can succeed while the other fails, and which order you do them in
determines what breaks:

- **Send to the queue first, then record that it was sent.** Crash in between those two, and
  the record still shows "not sent" — so it gets sent again, and the report goes out twice.
- **Record the message as sent first, then actually send it.** Crash in between those two, and
  the record says "sent" even though it never reached the queue — so the report is silently
  lost, with nothing showing that anything is wrong.

The tray removes that gap: writing the message to it *is* the moment it counts as built.
Sending it onward from there is a separate, always-safely-retryable follow-up step — the worst
case that step can still produce is explained next.

---

## 3. How duplicates are prevented

Every message has a **fingerprint** — an identity built from six things:

- which config
- which report type
- which slice of that config's scope (one account or alias for an unbundled config, or the
  whole scope tree for a bundled or no-scope config — §5)
- which time window
- which trigger
- which specific occurrence of that trigger produced it — a freshly minted identity for
  on-demand and PHT, or the exact slot it was scheduled for on a scheduled run

The same scheduled slot always reproduces the same identity, while a genuinely different
firing never does.

**The outbox tray refuses to hold two rows with the same fingerprint.** If the same message
somehow gets built twice — most often when a scheduled run and its own crash-recovery both
reach the same item (see §6 for what crash-recovery is) — the first row goes in, the second is
rejected, and the pod that lost sees "already there, nothing to do" and moves on. One copy,
ever. That's the real guarantee; there's no separate lock involved.

For example:

1. Pod A is working through a scheduled run's to-do list. It finishes building the message for
   item #47 and writes it to the outbox — then crashes a split second before it can mark that
   to-do item done.
2. Pod B's own watchdog tick (§6) notices the stale heartbeat and claims the run.
3. Since item #47's to-do row doesn't show "done" yet, pod B rebuilds it too and tries to write
   its own copy to the outbox.
4. Both copies carry the identical fingerprint (same config, report type, window, trigger,
   occurrence), so the outbox accepts pod A's row — it got there first — and rejects pod B's as
   a duplicate.
5. Pod B sees "already there, nothing to do," marks the to-do item done, and moves on.

No duplicate report, no lost work.

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

**Scheduled**

1. The scheduler fires (see `../scheduling/solution.md` for when and how often) — clustered, so
   exactly one pod picks up each firing.
2. That pod creates one tracking record for this run.
3. It checks a feature flag for that report type — off, and the run stops right there
   (`../scheduling/solution.md` §4).
4. Flag on, and it works through the matching report configurations in pages of around 500 at a
   time:
   - fetch the next page
   - pull all the payment-type data (accounts or aliases) for the whole page in one batch —
     not one query per config
   - decide how many messages each config produces (§5)
   - write those as to-do rows
   - for each one: build the message, write it to the outbox, mark it done
   - move to the next page
5. The relay is draining finished messages the whole time, in parallel (§2).

**On-demand**

1. A message arrives on the on-demand queue carrying a list of config ids.
2. The pod that picks it up mints a fresh identity for this request — there's no id supplied by
   the caller.
3. It creates a tracking record and runs the same resolve → build → outbox → done loop for just
   those configs.
4. It records the incoming message's id as handled and acknowledges the queue.

That minted identity is part of the fingerprint, which is why asking for the same report twice
on purpose produces two messages rather than being silently swallowed, and why an on-demand
message never collides with a scheduled one for the same window.

**PHT (the external balance push)**

1. A fixed-width text message arrives carrying account balances.
2. The pod parses it and works out the recipient from an identifier in the message.
3. It looks up that recipient's configuration — the one the scheduler never touches.
4. It mints a fresh identity for this request (same reason as on-demand).
5. It builds one message carrying those pushed balances and writes it to the outbox.
6. It records the incoming message as handled and acknowledges the queue.

---

## 5. What "one to-do item" means — the bundling rule

A single report configuration can turn into one message or many, depending on how it's set up:

| Configuration | Produces |
|---|---|
| **Bundled** | One message for the whole config — every payment type it has, each carrying all of that type's accounts or aliases, merged across every scope |
| **Unbundled** | One message per account or alias |
| **No scope attached** | One "just the configuration" message |

The to-do rows are written to match this — one row per message that will actually be produced.
That's also why the to-do list can only be written *after* a configuration's data has been
resolved: there's no way to know the count before then.

---

## 6. When a pod dies

**A scheduled run.** While a run is active, its pod updates a heartbeat on the tracking record
every so often. A dedicated watchdog job — also clustered — watches for tracking records whose
heartbeat has gone stale, meaning the owning pod died.

**How the watchdog works — it's not a notification system.** It's just another Quartz job,
registered on the same clustered scheduler already used for report timetables, firing on a
short interval — so only one pod's watchdog tick is ever scanning the tracking-record table at
a time, and that pod is the one that discovers a stale run, by querying for it itself. Nobody
assigns it anything. If it finds one, it's also the pod that claims it (a compare-and-swap, so
only one pod wins even if two ticks overlap) and resumes it immediately, in that same
execution. This deliberately reuses Quartz's clustering, but not Quartz's own `requestRecovery`
mechanism — a different, narrower tool already used elsewhere (a pod dying before a scheduled
run's tracking record even exists yet, `../scheduling/solution.md` §8). `requestRecovery`
re-fires one interrupted job once; it has no visibility into tracking records, pages, or
checkpoints, so only this watchdog — which does understand those — can resume a run already in
progress.

It resumes a claimed run in two parts:

1. **Re-fetches and rebuilds any to-do items that were still unfinished** — safe to redo, since
   the outbox fingerprint rule simply rejects anything already built before the crash.
2. **Keeps paging forward from wherever the dead pod left off**, exactly as normal processing
   would.

A run that keeps failing recovery past a set number of attempts is marked abandoned, with an
alert for a person. (One edge case — a pod dying before it even manages to create the tracking
record — is covered by the scheduler's own recovery, not this watchdog;
`../scheduling/solution.md` §8.)

**An on-demand or PHT message.** Much simpler — no watchdog needed. These arrived as queue
messages, so if a pod dies before finishing, the queue itself automatically redelivers the
message to another pod, which starts over. To avoid reprocessing a message that actually
finished just before the crash-and-redeliver, Commander records the id of every incoming
message it completes (the `ProcessedInboundMessage` table, §9) and skips any it's already seen.

---

## 7. How a to-do item ends

Every to-do item finishes in exactly one of these states:

| State | Meaning |
|---|---|
| **Built** | Its message is in the outbox. Done — this means *produced*, not *delivered* (see §8 for what can still happen to it after). |
| **Failed (poison)** | Tried several times, keeps failing — usually bad data. Terminal, and it **raises an alert**; the rest of the run carries on regardless. |
| **Obsolete** | During recovery, the account or scope this item was for turned out to have been genuinely removed. Logged, done, **not** an alert. |

(No flag-related state exists here — that's checked before a run starts and again at the relay,
never at build time; §8.)

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
- **Feature flags are checked twice — never in between — and both checks are terminal.**
  - Once per report type, before a run even starts — the scheduler's own check
    (`../scheduling/solution.md` §4). Off, and nothing is handed to the pipeline at all, no
    to-do items ever exist.
  - Once more, per report type, by the relay, right before it sends a given tray row. Off, and
    that row is done — not sent, not retried later, a message logged but not an alert, exactly
    as final as if it had been sent.

  Between those two points, a to-do item always gets built once it exists; there's no flag
  check at build time, so a to-do item always ends `Built` (§7) regardless of what the
  relay does with its row afterward — `Built` means the message was produced, not that it
  was delivered.
- **No ordering guarantee between messages.** The relay is sharded for throughput, so it makes
  no promise about the order messages land on the queue in. This is safe because every message
  is self-contained — it carries its own window and identity — and Executor is expected to
  process each one independently.
- **The reporting window always reflects when a run was scheduled for, not when it actually
  ran.** It's computed once, from the trigger's scheduled time, and frozen on the tracking
  record — so a delayed or recovered run still produces the exact window it was meant to.

---

## 9. The tables, in one line each

| Table | In plain words |
|---|---|
| **Run** | One row per triggered run — a schedule firing, or an accepted on-demand / PHT message. Holds the heartbeat and how far it's got; a report-type flag off at the start stops it here, terminal (§4, §8). |
| **WorkItem** | The to-do list: one row per message Commander intends to produce, plus its state and attempt count. |
| **Outbox** | The tray: finished messages waiting for the relay, one row per fingerprint, ever — sent if the report type's flag is on, done without sending if it's off (§8). |
| **ProcessedInboundMessage** | Ids of incoming queue messages already fully handled, so a redelivery doesn't cause a second run — this is the record-and-skip mechanism described in §6. |

---

## 10. What's still open

Not open design questions — three things to pin down before this is final:

- **How long to keep `ProcessedInboundMessage` ids** — read this off the actual redelivery /
  backout-queue configuration for the on-demand and PHT queues, rather than picking a number
  arbitrarily.
- **Make sure the Executor app dedupes incoming messages based on the fingerprint**, accepts two
  semantically-equal messages distinguished only by trigger metadata (§8), and is fine
  processing messages with no ordering guarantee (§8) — confirm this with the Executor team.
- **Decide how to guard against a PHT-only config id ending up in an on-demand request** —
  either enforce it at the source (the on-demand caller never sends one in the first place,
  straightforward since the same team builds that caller too) or defend inside Commander itself
  with a skip-and-log check rather than failing the whole request. The product owner's
  guarantee is that this shouldn't happen at all; pick whichever is worth building.
