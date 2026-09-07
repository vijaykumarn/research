Great, that's enough for me to design this properly. Let me first quickly explain "unit of work" in plain terms, then give you the architecture options.

## What "unit of work" means here

When the scheduled trigger fires, it's not one job — it's "make a report for every account that needs one," which could be thousands of accounts. If the pod dies after finishing 300 of 3,000, you don't want to start over from zero (wasteful) and you don't want to accidentally send the same report twice for those 300.

So I'm proposing: **one unit of work = one report for one account/config, for one point in time.** Each of these gets tracked individually in the database as "not started / in progress / done." That way, if a pod dies, another pod can look at the list and say "these 300 are done, skip them; these 2,700 are not, do them" — no duplicate work, no lost work.

This also gives us a natural way to answer your question #2: the combination of **which account/config + which report type + which date/period** uniquely identifies one specific report. We can use that combination as a "fingerprint" to detect "have I already sent this exact report?" — which solves the duplicate problem.

---

## Common design (applies to all 3 options below)

- **Quartz in "clustered mode"** — this is a built-in Quartz feature where all pods share the same Quartz tables in SQL Server, and Quartz itself guarantees only one pod actually fires a given scheduled trigger at a time. This satisfies your "only one pod acts on a scheduled trigger" requirement, for free.
- **A tracking table for scheduled runs**: when a scheduled trigger fires, the winning pod creates one "run" record and one row per account/config that needs a report (the "unit of work" list), each starting as PENDING. Processing then works through this list, marking each row DONE as it succeeds.
- **Recovery**: a small periodic check looks for runs that were left "stuck" (started, but no recent activity) and picks up the leftover PENDING rows — any pod can do this, not necessarily the one that started the run.
- **On-demand and PHT triggers**: since you're happy relying on standard queue retry behavior for these, they stay simple — process the message, and if something fails, the message goes back on the queue and is retried automatically. No special tracking table needed for these two.

Where the 3 options genuinely differ is **how we make sure a report never gets published to MQ twice.** As you asked, here they are with trade-offs.

---

### Option 1 — Simple: "Publish, then mark done" with a duplicate check

Process each report normally: build it, send it to MQ, then mark it DONE in the database. Before doing any of this, check the "fingerprint" (account + report type + date) against what's already been marked DONE — if it's already there, skip it entirely.

- **Risk window:** if the pod dies in the split second after sending to MQ but before marking DONE, a retry could send it again. This is rare but not impossible.
- **Pros:** Simplest to build, fastest, least infrastructure.
- **Cons:** Small chance of a duplicate message reaching Executor; Executor should ideally also check the fingerprint and ignore duplicates as a safety net.

### Option 2 — Safer: "Store the message safely first, then send it" (Outbox pattern)

Instead of sending to MQ directly, the pod does one safe database step: save the fully-built message into a database table (with the fingerprint as a uniqueness check) AND mark the work item DONE, both at once, safely. A separate small background step then reads unsent messages from that table and pushes them to MQ, marking them "sent" afterward.

- **Risk window:** shrinks a lot — the "decision to send" is now safely stored no matter when the pod dies. There's still a tiny window in the final "send to MQ" step itself, but it's much smaller and easier to make safe (e.g., check "sent" flag before sending).
- **Pros:** Much safer against duplicates and lost messages; the risky moment is isolated to one small, easy-to-protect step.
- **Cons:** More moving parts (an extra table, an extra background step, some cleanup of old sent records over time); slightly more delay between "report ready" and "message on MQ" (usually seconds).

### Option 3 — Most robust: Same as Option 2, but the "send it to MQ" part is its own clean, reusable component

Structurally the same as Option 2, but instead of each trigger (scheduled/on-demand/PHT) having its own bit of "send to MQ" logic, all three funnel into one shared, well-tested internal component (could later even become its own small service) whose only job is: "take unsent messages from the table, send them to MQ safely, mark them sent." 

- **Pros:** One place to get the tricky "don't duplicate" logic right, instead of three; easier to add new report types or trigger sources later without touching publish logic; easier to monitor ("how many messages are waiting to be sent right now?").
- **Cons:** More upfront design/structuring effort; only worth it if you expect this service to grow (more triggers, more report types) — otherwise it's extra complexity for the same safety as Option 2.

---

**My recommendation given what you've told me:** Option 2 is the sweet spot for most enterprise reporting services — noticeably safer than Option 1 for not much extra cost, and simpler than Option 3. Option 3 is worth it if Commander is expected to keep growing (more triggers, more report types, maybe other consumers besides Executor someday).

Want me to go deeper into one of these — e.g., draw out the database tables and the step-by-step flow, or a diagram?