# Commander — Data Retrieval Solution

This is the reference for how Commander turns a set of report configurations into everything
needed to build their messages — every property, the whole account/alias tree, the recipient.

> Visual: `data-fetch-flow.drawio` (draw.io / diagrams.net) — the six staged reads, what each
> read keys off, and the recovery tri-state.

> Building or reviewing the code? `solution_v02.md` has the exact SQL, the schema addition,
> and the full reasoning against the alternatives this document leaves out on purpose, to stay
> readable.

---

## 1. What this layer is responsible for

Given a set of report configurations — however they were picked — resolve everything the
pipeline needs to build their messages: every property on the config itself, its whole scope
tree (payment types, and under each, either accounts or aliases), and its recipient.

It doesn't decide which configs are due (that's scheduling) and it doesn't build or send a
message (that's the pipeline). It turns "here are some config ids" into "here's everything
about them" — nothing more.

It's read-only and stateless. Many pods call it at once, with no coordination needed between
them, and "the data as of when this call reads it" is good enough — a config edited mid-run
just gets picked up on the next run.

---

## 2. The shape: six round trips, however wide the page

Resolving a page of 500 scheduled configs, a short on-demand list, or one PHT config all cost
the same six database round trips — one per level of the hierarchy:

1. config rows
2. scopes
3. payment types
4. accounts
5. aliases
6. recipients

Everything after that is stitched together in memory, with no database involved, into one
fully-resolved structure per config.

This "six round trips, then assemble" shape is deliberate: the cost scales with how many
*pages* get resolved, not with how many configs there are or how wide any one config is.
Resolving one config at a time — a query set per config — would cost roughly five round trips
× however many configs, which at 10,000 configs is about 50,000 round trips. Batching by page
instead brings a 5,000-config run down to around 10 pages, roughly 60 round trips total.

---

## 3. Two entry modes: fresh resolution, and recovery

Everything above is the shared core. On top of it, there are two ways to call in:

**`resolvePage` — fresh resolution**, for a set of configs that haven't been resolved before
this call. The config set can come from three places:

- **Scheduled**, a page at a time: the next batch of active configs for a given report type
  and frequency, fetched in order by id, picking up wherever the last page left off. Stable
  even while configs are being added or edited underneath it — a row inserted behind the
  cursor just isn't seen by this run.
- **On-demand**, a caller-supplied list of config ids — tens of them, not thousands.
- **PHT**, indirectly: first resolve which recipient a pushed message is for (a separate small
  lookup by an external identifier, `Agreement → AgreementVersion (active) → AgreementScope`),
  then resolve that one recipient's one active PHT config the normal way. See §6 for what
  happens to the resolved data after.

**`reResolve` — recovery**, for `WorkItem`s the pipeline already created and now needs to
check on again after a crash. This is a genuinely different question from fresh resolution:
not "what configs are due," but "for these specific pieces of work the pipeline already
committed to, do they still exist?" See §5.

---

## 4. What comes back for each config

A fully-typed structure, not a loose property map: every column on the config, named
individually; its recipient; and its scope tree underneath — payment types, and under each,
either its accounts or its aliases, never both. A config with no scope at all still resolves
cleanly, just with an empty tree instead of an error.

Both the config's internal id and its business-facing config id are carried through, because
different callers need different ones — the pipeline's own tracking keys on the internal id
(stable, never reused), while an on-demand caller addresses configs by the business one.

---

## 5. Recovery's tri-state answer

When the pipeline's recovery re-checks a `WorkItem` it already has a row for, resolving the
same way fresh resolution does isn't quite enough — recovery needs to know not just *what* the
current data is, but *whether that work is still relevant at all*. So `reResolve` answers with
one of three outcomes for each piece of work:

- **Found** — the data is still there; here it is. The pipeline rebuilds and publishes.
- **Confirmed absent** — the walk succeeded and this piece of work is genuinely gone (the
  account or alias was removed, or the whole config was deleted or deactivated since the
  original run). The pipeline marks that `WorkItem` obsolete — not a failure, not alerted.
- **Query failed** — a lookup threw or timed out. This says nothing about whether the work
  still exists, so the pipeline just retries, the same as any other transient failure.

That distinction matters: a config going inactive and a database hiccup must never be confused
with each other. A hiccup that got mistaken for "gone" would quietly throw away real work; a
"gone" that got mistaken for a hiccup would keep retrying forever. So recovery's own lookup of
which configs to check deliberately does **not** filter by whether a config is still active —
unlike scheduled and on-demand resolution, which only ever look at active configs — precisely
so it can see and confirm the difference between "still there" and "genuinely deactivated."

New work that would exist today but didn't exist at the time of the original run is ignored —
recovery only ever answers for pieces of work the pipeline already committed to, it doesn't go
looking for more.

---

## 6. Matching a to-do item back to its data — the scope_key

Each `WorkItem` the pipeline creates needs a way to say, precisely, which piece of a config's
resolved tree it's for — so recovery can look that exact piece up again later. That identity is
a short, readable string:

| The to-do item is for | Looks like | Recovery looks for |
|---|---|---|
| One account (unbundled) | `ACC\|paymentType\|clearingNumber\|accountNumber` | that account, under a payment type of that kind |
| One alias (unbundled) | `ALS\|paymentType\|aliasId` | that alias, likewise |
| The whole config (bundled) | `BND` | the config row, still active — found means the *entire* current tree, every payment type, rebuilt in full |
| The config itself (no scope) | `CFG` | the config row, still active |

Deterministic and human-readable on purpose — an engineer can read one back and know exactly
what it's pointing at, without decoding anything.

There's only ever **one** `BND` to-do item per bundled config, never one per payment type — a
bundled config produces exactly one message covering everything it has
(`../message-pipeline/solution.md` §5), so that's the natural unit of work for it too. `BND`
and `CFG` both resolve the same way underneath (is the config still active), but stay distinct
markers because they're different report shapes — one carries a full scope tree, the other
never had one.

---

## 7. PHT's balance merge

The external balance-push flow ends with one extra step beyond the normal walk: the pushed
message carries account balances that need to land on the resolved accounts. This is a pure,
in-memory step, not another database call — each resolved account is matched against the
pushed balances by clearing number and account number. A match carries the pushed balance
forward; an account the push didn't cover is dropped from that message, since a PHT push only
ever reports on the accounts it actually carries. Aliases are untouched.

---

## 8. Keeping one wide config from blowing up a page

Page *count* is controlled directly (how many configs per page), but page *width* isn't — one
very wide bundled config, with thousands of accounts under a single payment type, can blow up a
page's memory footprint on its own, regardless of how few configs share that page. Two guards:

- **A per-config account ceiling.** A config that resolves to more accounts than a configured
  limit is almost certainly a misconfiguration (or would exceed the message-queue's size limit
  anyway). It isn't built — it's logged, alerted, and handed back to the pipeline as a poison
  item, the same as any other bad-data config, not silently truncated.
- **A hard heap-safety cap** on the total rows accumulated while resolving one page, independent
  of the per-config ceiling, so a page is never allowed to grow unbounded even if that ceiling
  is set generously.

A related guard sits in the assembler: a payment-type assignment is only ever supposed to carry
accounts *or* aliases, never both. If one somehow does, that single config is isolated as a
poison item the same way — the rest of the page keeps assembling around it. One malformed
config shouldn't deny service to the hundreds of well-formed ones sharing its page. The one
thing worth watching operationally: a single config tripping this is almost certainly bad data,
but a *high rate* of it across a page or run is more likely a bug in the reads themselves, and
should be alerted on louder than an ordinary poison item.

---

## 9. A few reliability details worth knowing

- **Each of the six reads fails independently.** A failed read is logged and rethrown, never
  swallowed or partially returned — the pipeline's own retry machinery owns what happens next.
  A transient failure on one read doesn't take down reads that would have succeeded.
- **The fan-out reads (payment types, accounts, aliases) use a database feature to hand in a
  whole batch of ids at once**, since a page's id set can run into the thousands — well past
  what a plain list of query parameters can hold. This needs one small, general-purpose
  addition to the database schema (§8, `solution_v02.md`) — not report-specific, one type for
  every fan-out read.
- **This design assumes the database's own indexes already support these access patterns** —
  keyset paging by id, and looking children up by their parent's id at every level. That's a
  prerequisite to verify before build, not something this layer adds.

---

## 10. What's still open

- **Confirm the one schema addition is acceptable** — everything here assumes it; a fallback
  exists (chunking id sets into smaller batches) but costs the flat per-page round-trip count.
- **Verify the required indexes exist** on the existing schema — without them, the "cost scales
  with pages, not configs" property partly evaporates.
- **Sign off the `scope_key` grammar (§6) jointly with the pipeline team** — it also feeds
  identity fields on their side.
- **Set the per-config account ceiling and the hard row cap (§8) from real data** — the current
  defaults are illustrative, not measured.
- **Does recovery ever need refreshed recipient data**, or only the account/alias leaves? If
  only leaves, recovery can skip that read entirely.
