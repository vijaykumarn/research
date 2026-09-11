# Commander — Data Retrieval: Architecture Options (v01)

Three distinct approaches to the data-retrieval layer: given a set of `ReportConfig`s, produce
a fully-resolved structure per config (own properties, resolved scope hierarchy, recipient)
for the message-pipeline's assembly step. Reflects `data-retrieval/goal.md` only — this layer
owns resolution, nothing else (scheduling and the pipeline are siblings, not inputs here).

---

## Settled foundation — applies to every option below

**The three entry points share one core; only config-set selection differs.**
- **Scheduled:** `WHERE report_type = ? AND frequency = ? AND is_active = 1 AND Id > :lastId
  ORDER BY Id` — **keyset pagination on the surrogate `Id`**, not `OFFSET`/`FETCH`. Stable
  under concurrent insert/update (a row inserted behind the cursor is simply never seen by
  this run; nothing shifts under an in-flight page), and every page is an equally cheap
  index seek — no growing-offset scan cost as the run progresses. Satisfies requirement 3
  directly; no option below changes this.
- **On-demand:** a small, bounded `WHERE ConfigId IN (...)` (tens of business keys) — no
  batching concerns at this level regardless of option.
- **PHT:** `Agreement → AgreementVersion(ACTIVE) → AgreementScope` resolves one `Recipient`
  from the external engagement identifier, then that recipient's single active PHT config.
  This is a single-row lookup, not a batch — it's the *entry point*, not part of the shared
  core. Once resolved, the one config id it produces is handed to the same shared core as the
  other two paths, then the balance-merge step (below) runs as a pure post-processing step.

**Two modes on the shared core, not one.** The pipeline uses this layer two different ways,
and they are genuinely different query shapes, not the same call with a different filter:

- **Fresh resolution** (`resolvePage`) — given a set of config ids (a page, a list, or one PHT
  config), walk the whole hierarchy top-down and return a full `ResolvedConfig` per config.
- **Recovery re-resolution** (`reResolve`) — given a set of *already-known* scope keys
  (leaf-level identities pulled from existing `WorkItem` rows), fetch fresh data for exactly
  those leaves and report, per key, **found** / **confirmed-absent** / **query-failed**
  (requirement 10). Critically, this is a **scattered, leaf-keyed lookup**, not a top-down
  walk from `ReportConfig` — the scope keys being re-checked may belong to many different
  original configs and pages, and recovery doesn't need to re-verify that the config or its
  scope still link to that leaf, only whether the leaf itself is still there and what its
  current data is. Designing the shared core as independently-callable per-level queries
  (rather than one monolithic config-rooted walk) is what makes this second mode possible
  without a second, parallel implementation — see the options below for how each handles this.

**Recipient batching (requirement 4)** is a non-differentiator: collecting a page's distinct
`MessageRecipientId`s and querying `Recipient` in one batch is trivial at page scale (at most
a few hundred ids — never near SQL Server's parameter cap) and every option below does it the
same way, so it isn't discussed per-option.

**Assembly is pure and DB-independent (requirement 5).** Whatever shape the query layer
returns, a separate, dependency-free function turns it into the output tree. This function
takes plain DTOs/records in and returns `ResolvedConfig`s out — unit-testable with hand-built
fixtures, no test database required. Illustrative shape:

```java
public record ResolvedConfig(
    long id, String configId,              // requirement 8: both carried through
    Map<String, Object> properties,         // requirement 6: every ReportConfig column, opaque to this layer
    Recipient recipient,
    List<ResolvedScope> scopes              // empty list, not null, for a zero-scope config
) {}
public record ResolvedScope(long agreementScopeId, List<ResolvedPaymentType> paymentTypes) {}
public record ResolvedPaymentType(
    long paymentTypeAssignmentId, String paymentType,
    List<Account> accounts, List<Alias> aliases   // exactly one of these is non-empty
) {}
```

**Zero-scope configs (requirement 7)** fall out of the same mechanism as any other config with
no matching child rows — the scope-level query simply returns nothing for that config id, and
the assembler produces an empty `scopes` list. No special-casing needed in any option.

**Read consistency (§4 of the brief).** No snapshot isolation, no explicit transaction
wrapping the per-level queries — read-committed is sufficient, since a config/scope change
mid-run is picked up on the next run by design. This simplifies every option: there's no need
to correlate multiple queries to one consistent point in time.

**Requirement 11 — the parameter-limit problem, and the "new DB object" question it raises.**
The first level (config selection) never needs more than a range predicate or a few hundred
ids — never close to SQL Server's ~2,100 parameter cap. The problem is every level *below*
that: a page of 500 configs can easily produce several thousand `AgreementScope` ids, and
`PaymentTypeAssignment WHERE AgreementScopeId IN (...)` with one parameter per id blows past
the cap. Two ways to handle it, independent of which option is chosen:
- **Table-Valued Parameter (recommended):** a new user-defined table type in the `CAMT` schema
  (e.g. `CREATE TYPE dbo.IdList AS TABLE (Id BIGINT NOT NULL PRIMARY KEY)`), passed as a single
  parameter and joined against — `... JOIN @ids i ON t.AgreementScopeId = i.Id`. **This is a
  new database object** — the brief asks this be surfaced explicitly, so: one small,
  general-purpose type, not a report-specific table, reusable at every level and by the
  recovery path's leaf lookups too.
  Any option below can use this
  once, at every level, without further schema additions.
- **Chunked `IN` clauses (fallback, no new object):** split an id set over ~2,000 into
  sub-batches and issue multiple queries per level, merging results in the adapter. Avoids
  touching the schema, at the cost of a variable, run-dependent number of extra round trips
  exactly when a page happens to be "wide" (many scopes/accounts per config) — the worst case
  works against requirement 1's per-page cost goal. Worth keeping as the answer only if adding
  a TVP type is genuinely off the table for this schema.

The rest of this document assumes the TVP is available; each option would need re-costing
under the chunked fallback if it isn't.

---

## Option A — Per-level batch queries, application-orchestrated

**Mechanism.** One JDBC round trip per hierarchy level, each filtered by the previous level's
result via the TVP: configs → scopes → payment-type-assignments → accounts → aliases →
recipients. Six flat, narrow queries per page (five for a zero-recipient-batching case is not
possible since recipient is always resolved, so six is the fixed count). Each returns a simple
flat DTO list; the pure assembler (above) groups them into the tree by parent id.

**Requirement fit.**
- **Req 10 (tri-state recovery):** the natural fit. Recovery's leaf-keyed lookups map directly
  onto *one* of these level queries — re-checking a set of account-level scope keys is just
  the accounts query run against exactly those ids, no walk from `ReportConfig` needed. A
  level query that succeeds but returns fewer rows than requested ids gives **confirmed-absent**
  for the missing ones; a level query that throws gives **query-failed** for that batch, without
  affecting other levels' results in flight.
- **Hexagonal fit:** the cleanest of the three. Each level is a small, independently testable
  repository method; the adapter is plain `JdbcTemplate`-style SQL, no SQL Server-specific
  multi-resultset handling.

**Trade-offs.** Six round trips per page rather than one. At the stated page size (a few
hundred configs) and volumes (thousands of configs → tens of pages per run), this is a few
hundred round trips for a whole scheduled run — not per-config, so requirement 1 is satisfied;
the cost is a small, fixed multiplier per page, not a growth curve. This is the option closest
to what `how-it-works.md`'s "one set of bulk queries pulls all the data... at once" line
already implies.

---

## Option B — Single stored procedure, multiple result sets

**Mechanism.** One `usp_ResolveConfigPage(@ids dbo.IdList READONLY)` call returns the same six
flat result sets as Option A, produced server-side in one execution — the procedure holds
intermediate id sets (scope ids, payment-type ids, …) in local temp tables internally, so only
one TVP crosses the wire and only one JDBC call is made per page (read via
`CallableStatement` + repeated `getMoreResults()`). The pure assembler downstream is unchanged
from Option A.

**Requirement fit.**
- Satisfies req 1/11 with the fewest round trips of the three — one call per page regardless
  of hierarchy depth.
- **Req 10 is weaker here.** The procedure is naturally shaped around "start from config ids,
  walk down" — recovery's scattered, leaf-keyed re-checks don't fit that shape well. It would
  need a second procedure (or a mode flag branching internally in T-SQL) built around "start
  from leaf ids directly," duplicating logic across two procedures instead of reusing one set
  of composable queries. A partial internal failure (one of the six internal SELECTs erroring)
  is also harder to expose to the caller with per-level granularity than in Option A, since
  the whole call is one unit as far as JDBC is concerned unless the procedure is written to
  catch and report per-level status itself — extra T-SQL complexity, not free.

**Trade-offs.** Fewer round trips, at real operational cost: a stored procedure is a bigger new
database object than the TVP type alone — it needs its own migration/versioning, DBA review,
and its SQL logic is materially harder to unit test than Option A's (the domain assembler
stays pure and testable either way, but the six-way T-SQL walk itself typically needs an
integration test against a real SQL Server instance rather than a unit test). Hexagonal
layering is preserved at the port/domain boundary — the adapter just gets thicker.

---

## Option C — Single flattened join query per page

**Mechanism.** One wide `SELECT` per page: `ReportConfig` `LEFT JOIN` down through
`AgreementScope`, `PaymentTypeAssignment`, `AccountAssignment`/`AliasAssignment`, and
`Recipient`, filtered by the page's config ids via the TVP. `LEFT JOIN`s throughout so a
zero-scope config still produces one row (with nulls from `AgreementScope` on). Returns one
denormalized flat rowset — one row per (config, scope, payment-type, account-or-alias) leaf —
with the config's own properties and recipient fields repeated across every matching row. The
assembler groups by config id and de-duplicates the repeated parent-level columns as it
builds the tree.

**Requirement fit.**
- True single round trip, and — unlike Option B — no stored procedure: plain SQL, a direct fit
  for the brief's "no ORM, plain JDBC" constraint, and the only new database object needed is
  the TVP type shared with the other options.
- **Req 10 is the weakest of the three.** A single query is a single failure domain: it either
  returns full data or throws for the whole batch, so there's no natural per-item
  confirmed-absent-vs-query-failed split *within* one call the way Option A's independent level
  queries give for free. (A requested id simply missing from the joined result *does* read as
  confirmed-absent as long as the query succeeds — so the distinction isn't lost, just coarser:
  one failure domain covering every level at once, rather than per-level isolation.) Retrofitting
  this for recovery's leaf-keyed lookups also means a second, differently-filtered join — the
  same "two shapes of query, not one" duplication concern as Option B, without B's round-trip
  advantage to offset it.

**Trade-offs.** Significant wire and row-mapping bloat: a bundled config with hundreds of
accounts repeats its own ~20 properties and the recipient's fields on every one of those rows.
For a page containing even a handful of very wide configs, this can dominate page processing
time unevenly — a "hot page" problem the other two options don't have, since their config- and
recipient-level data is fetched exactly once regardless of how many leaves a config has. The
five-way `LEFT JOIN` (with variable cardinality at each level) is also the hardest of the three
query shapes for the SQL Server optimizer to plan predictably and for a DBA to tune later.

---

## Comparison

| | A — per-level batch queries | B — stored procedure, multi-resultset | C — single flattened join |
|---|---|---|---|
| Round trips per page | ~6 | 1 | 1 |
| Req 10 (tri-state recovery fit) | Natural — level queries are independently reusable for leaf-keyed lookups | Weak — needs a second procedure or an internal mode branch | Weakest — single failure domain, needs a second query shape |
| New DB objects | TVP type only | TVP type **+** stored procedure | TVP type only |
| Wire volume | Narrow, no duplication | Narrow, no duplication | Denormalized — parent data repeated per leaf row |
| SQL testability | Each level unit-testable in isolation; adapter is plain JDBC | Six-way T-SQL walk needs integration tests; domain assembler still pure | One big join needs integration tests; assembler's de-dup logic is more intricate |
| Query-plan predictability | High — each query is a targeted seek | Same queries as A, server-side | Lower — 5-way variable-cardinality join |
| Hexagonal fit | Cleanest — thin adapter | Preserved, thicker adapter | Preserved, thicker assembler |

---

## Recommendation

**Option A.** Requirement 10's recovery re-resolution is a real, load-bearing part of this
layer's contract — not an edge case — and it specifically wants the ability to run a single
targeted leaf-level query without re-walking a whole config hierarchy. Option A gives that for
free, by construction, because its six queries are already independent and composable; B and C
would each need a second, parallel query shape to serve recovery well, which is real
duplicated logic and surface area neither option's round-trip savings clearly justifies at
this volume (a few hundred round trips for a whole multi-thousand-config run is a small,
bounded cost, not a scaling risk). Option A also keeps every requirement's implementation in
plain, unit-testable application code, consistent with the brief's "no ORM, plain JDBC"
constraint and its hexagonal-layering expectation.

Option B remains available if profiling a real run later shows round-trip latency — not query
*count* per config, which is already handled — is an actual bottleneck; that's a narrower,
evidence-backed reason to take on a stored procedure's operational cost, not a starting
assumption. Option C is the weakest fit given req 10 and the wire-bloat risk on wide configs,
and I wouldn't recommend it unless minimizing new database objects turns out to matter more
than every other requirement here — worth naming as the minimalist fallback, not a contender.

---

## Open items for the next pass

- Confirm the TVP user-defined table type is an acceptable schema addition to `CAMT` — this is
  the one new database object every option (including the recommendation) depends on.
- Exact page size / round-trip count to validate empirically once real config-to-scope-to-leaf
  ratios are known (the "500 configs → thousands of scope ids" figure above is illustrative).
- The recovery re-resolution API shape (`reResolve(Set<ScopeKey>)`) needs to be reconciled with
  exactly what a `ScopeKey` encodes in the pipeline's `WorkItem.scope_key` column, so the two
  documents agree on the identity format being passed across the boundary.
- Whether the config-level query's full property pass-through (requirement 6) is best modeled
  as an untyped property map (as sketched above) or a generated/explicit DTO with one field per
  `ReportConfig` column — a maintainability trade-off, not an architectural one, deferred here.
- Confirm with the pipeline side whether recipient data ever needs to be included in recovery
  re-resolution, or whether recovery only ever needs to refresh leaf (account/alias) data.
