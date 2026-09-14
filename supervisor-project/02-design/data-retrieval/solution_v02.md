# Commander — Data Retrieval: Design (v02)

The recommended design. Given a set of `ReportConfig`s, produce a fully-resolved
`ResolvedConfig` per config (all properties + scope hierarchy + recipient) for the pipeline's
message-assembly step.

It is **staged per-level batch reads + a pure in-memory assembler** — the shape the legacy
`bikili` implementation already runs in production, hardened with batched recipient lookup, an
explicit recovery mode, and a stated `scope_key` contract. `archive/solution_v01.md`'s Option A
is the same choice; this document commits to it and fills in the parts left open there.

---

## 1. Shape at a glance

Per page (or per on-demand list, or per PHT config), a **fixed six round trips**, none of them
per-config:

```
1. config page      keyset:  WHERE report_type=? AND frequency=? AND is_active=1
                             AND Id > :lastSeenId  ORDER BY Id  FETCH NEXT :pageSize
2. scopes           plain IN (:configIds)                 — bounded by page size
3. payment types    TVP (scope ids)                       — can fan out
4. accounts         TVP (payment-type-assignment ids)     — can fan out
5. aliases          TVP (payment-type-assignment ids)     — can fan out
6. recipients       plain IN (:distinct MessageRecipientIds)   — bounded by page size
```

Then a **pure, DB-free assembler** groups the flat rows by parent id into a
`List<ResolvedConfig>`. A 5,000-config scheduled run at page size 500 is ~10 pages ≈ **60
round trips total** — flat in the number of pages, independent of how wide any config is
(requirement 1).

> Visual: `data-fetch-flow.drawio` (draw.io / diagrams.net) — page 1 the six staged reads,
> page 2 what each read keys off, page 3 the recovery `reResolve` tri-state.

---

## 2. The shared core and its two entry modes

**The shared core is Levels 2–6** (scope → payment type → account/alias → recipient). Level 1
— selecting *which* config rows — has **three** variants, one per caller:

| Caller | Level-1 query | `IsActive` filter |
|---|---|---|
| Scheduled | keyset: `WHERE ReportType = ? AND ReportFrequency = ? AND IsActive = 1 AND Id > :lastSeenId ORDER BY Id FETCH NEXT :pageSize` | yes — never start work for an inactive config |
| On-demand | `WHERE ConfigId IN (:configIds) AND IsActive = 1` — `ConfigId` is the business key | yes |
| **Recovery** | `WHERE Id IN (:ids)` — surrogate `Id`s taken from the `WorkItem` rows, **no `IsActive` filter** | **no** — recovery must be able to *see* that a config went inactive |

Everything below Level 1 is identical for all three.

### `resolvePage(configIds) → List<ResolvedConfig>` — fresh resolution

Scheduled / on-demand Level-1 (above) + Levels 2–6. The whole top-down walk. For the scheduled
path the caller has already run Level 1 and passes the `ReportConfigRow`s in.

### `reResolve(Set<WorkItemRef>) → Map<WorkItemRef, ResolveResult>` — recovery

The pipeline's recovery (`../message-pipeline/solution_v08.md`) re-drives *existing*
`WorkItem` rows; it does **not** rediscover membership. Each `WorkItemRef` is
`(config_id, scope_key)` where `config_id` is the surrogate `Id` (§6). Per ref, a **tri-state**
(requirement 10):

| Result | Meaning | Pipeline does |
|---|---|---|
| `FOUND(data)` | the leaf still resolves under that config; here is its current data | rebuild + publish |
| `CONFIRMED_ABSENT` | the walk succeeded and this `scope_key` is genuinely gone (the leaf was removed, or the whole config went inactive/was deleted) | mark the `WorkItem` `OBSOLETE` |
| `QUERY_FAILED` | a query threw / timed out — no statement about presence | retry (→ `FAILED_POISON` if it keeps failing) |

Algorithm:

1. Group the `WorkItemRef`s by `config_id`.
2. Run the **recovery Level-1** query (`Id IN (:ids)`, no active filter) for the distinct
   config ids, then Levels 2–6 for the configs that came back.
   - If a level query **throws** for a batch → `QUERY_FAILED` for every `WorkItemRef` of every
     config in that batch; other batches are unaffected (staged reads are per-batch, so the
     failure is naturally scoped).
3. **Per config, branch before the presence check:**
   - **Config row not returned** (deleted) **or `IsActive = 0`** (deactivated since the
     original run) → `CONFIRMED_ABSENT` for *every* `WorkItemRef` under it. Log the reason
     (`config-deleted` / `config-deactivated`) distinctly from a leaf removal, for
     observability — but the result handed back is the same, because the pipeline's action
     (`OBSOLETE`) is the same. This is consistent with BR7: a config change mid-run needn't be
     reflected in that run, and a config that is no longer active has nothing left to report.
   - **Config active** → for each of its `WorkItemRef`s, look up the `scope_key` in the
     freshly-resolved `ResolvedConfig`: present → `FOUND` with the fresh sub-tree; absent →
     `CONFIRMED_ABSENT`.

New leaves that appeared under a config since the original run are **ignored** — recovery only
answers for `scope_key`s that already have `WorkItem` rows.

---

## 3. The `scope_key` contract

`WorkItem.scope_key` (`VARCHAR(200)` in `solution_v08.md`) is the identity `reResolve` matches
on, so its grammar must be pinned and shared across the two documents. Proposed:

| Work item kind | `scope_key` | `reResolve` looks for |
|---|---|---|
| Unbundled — account | `ACC\|{paymentType}\|{clearingNumber}\|{accountNumber}` | that account under a payment-type assignment of that type, in the config's tree |
| Unbundled — alias | `ALS\|{paymentType}\|{aliasId}` | that alias, likewise |
| Bundled — whole config | `BND` | the config row itself still active; `FOUND` returns the *entire* current tree — every payment type, every account/alias under it, merged across scopes (the whole bundle is rebuilt in full — confirmed against the legacy `bikili` assembler, `ReportMessageAssembler`/`PaymentTypeGrouper`: a bundled config is exactly one `ReportMessage`, never one per payment type) |
| Config-only (zero scope) | `CFG` | the config row itself still active |

Deterministic, human-readable, fits well under 200 chars. **This grammar needs a joint
sign-off with the pipeline side** (it also feeds `UQ_WorkItem_Identity` and the `Outbox`
`scope_key` column).

**`BND` and `CFG` share a marker shape (no parameters) and the same presence check** — is the
config still active — but stay distinct, because they're different report shapes: `BND` always
carries a resolved scope tree, `CFG` never does. There is only ever **one** `BND` `WorkItem` per
bundled config, matching the one-message-per-config unit of work — not one per payment type, as
an earlier draft of this grammar assumed.

---

## 4. The staged reads

### Level 1 — config selection (three variants — see §2)

- **Scheduled — keyset pagination on the surrogate `Id`**: `WHERE ReportType = ? AND
  ReportFrequency = ? AND IsActive = 1 AND Id > :lastSeenId ORDER BY Id FETCH NEXT :pageSize`.
  Stable while `ReportConfig` is being written concurrently (a row inserted behind the cursor
  is simply never seen by this run; nothing shifts under an in-flight page), and every page is
  an equally cheap index range scan — no growing-offset cost (requirement 3).
- **On-demand**: `WHERE ConfigId IN (:configIds) AND IsActive = 1` — `ConfigId` is the
  business key the caller supplies (tens of ids, no batching concern).
- **Recovery**: `WHERE Id IN (:ids)` — surrogate `Id`s from the `WorkItem` rows, **no
  `IsActive` filter**, so `reResolve` can tell a deactivated config from a removed leaf (§2).
  The id set is bounded by how many distinct configs a recovery batch touches; a recovery
  batch is pipeline-sized (≤ `page-size` distinct configs), so a plain `IN` is safe — but
  apply the **same `in-list-guard` ceiling** as the scope query below, so an oversized batch
  fails fast rather than being sent as one giant `IN`.
- `NEVER`-frequency configs are excluded structurally by the scheduled predicate
  (`frequency = ?` never equals `NEVER`).

### Level 1 → 2 — scopes, plain `IN`

`ReportAgreementScope JOIN AgreementScope` filtered by `ReportConfigId IN (:configIds)`. The
id set here is bounded by the page size, so a plain `IN` is safe. Keep a **guard assertion**
(fail fast if the set exceeds a safe ceiling, e.g. 2,000) — a violated invariant means a
caller is assembling for too many configs at once.

### Levels 2 → 3 / 3 → 4 — payment types, accounts, aliases, via **TVP**

These fan out — a page of 500 configs can produce several thousand scope ids and more
assignment ids, past SQL Server's ~2,100-parameter cap. Bind the id set as a **table-valued
parameter** and join against it:

```sql
SELECT ... FROM CAMT.PaymentTypeAssignment pta
JOIN @ids i ON i.Id = pta.AgreementScopeId
OPTION (RECOMPILE)
```

Accounts and aliases are two separate queries, both keyed by the payment-type-assignment ids.

**Add `OPTION (RECOMPILE)` to the TVP queries.** SQL Server estimates a table-valued parameter
at **1 row** unless the statement is recompiled with the actual value in hand. A TVP that
really holds 3,000 ids planned as if it held 1 tends to get a nested-loop plan that degrades
badly at volume. `OPTION (RECOMPILE)` lets the optimiser see the real row count; these queries
run ~3 per page (tens of times per run, not thousands), so a per-statement recompile is cheap
insurance.

One caveat so this isn't over-sold: `RECOMPILE` fixes the **row count**, not
value-distribution statistics — a TVP carries no histogram, so the optimiser still has no idea
*which* ids are in it. For the query shapes here (a straight `JOIN` on an id list, no second
selective predicate) that doesn't matter — cardinality is what drives the plan. It would start
to matter if a future TVP-backed query added another filtering predicate alongside the join; at
that point materialising the ids into a `#temp` table with a primary key (which *does* get
real statistics) becomes the better tool. For this design, `OPTION (RECOMPILE)` is the
recommendation; `#temp` and trace flag 2453 are the fallbacks if a plan-cache purist objects.

### Level 6 — recipients, plain `IN` (requirement 4)

Collect the page's **distinct** `MessageRecipientId`s, one `SELECT ... WHERE Id IN (...)`. A
page has at most `pageSize` distinct recipients — never near the parameter cap. The recipient
is attached to each `ResolvedConfig` by the assembler; it is **not** a downstream per-config
lookup (this is the one place the legacy implementation had a hidden N+1).

### Error handling

Each stage runs in its own try/catch that logs `(reportType, frequency, lastSeenId, stage)`
and **rethrows** — never swallows, never partially returns. A failed stage aborts that
page/batch cleanly; the pipeline's `WorkItem`/attempt-count machinery owns the retry.

### Required indexes — the design's per-page cost depends on these

The "flat cost per page" property only holds if the `CAMT` schema carries the indexes these
queries seek on. **Verify they exist before build** (they are on the pre-existing schema, not
something this layer adds):

| Query | Needs |
|---|---|
| Level 1 config page | Composite on `ReportConfig (ReportType, ReportFrequency, IsActive, Id)`. Without it, each page scans the table and re-sorts — the keyset advantage (no growing-offset cost) partly evaporates. |
| Level 1 on-demand | Index on `ReportConfig (ConfigId)` (likely already unique). |
| Level 1→2 scopes | Index on `ReportAgreementScope (ReportConfigId)`; PK on `AgreementScope (Id)`. |
| Level 2→3 payment types | Index on `PaymentTypeAssignment (AgreementScopeId)`. |
| Level 3→4 accounts / aliases | Index on `AccountAssignment (PaymentTypeAssignmentId)` and `AliasAssignment (PaymentTypeAssignmentId)`. |
| Level 6 recipients | PK on `Recipient (Id)`. |
| PHT recipient resolution | Indexes supporting `Agreement.EngagementId` → `AgreementVersion (AgreementId, Status)` → `AgreementScope`. |

If any are missing, that is a schema-migration prerequisite for this layer to meet its
performance goals — surface it alongside the TVP-type question (§8).

### Bounding page width

Page *count* is controlled by `page-size`; page *width* is not. One very wide bundled config
(thousands of accounts under a payment type) can, on its own, make a page's account result set
and the heap holding that page's resolved trees blow up regardless of how few configs the page
has. Two guards:

1. **Per-config account ceiling** (`commander.read.max-accounts-per-config`, e.g. 25,000).
   After assembly, a config whose resolved account count exceeds this is almost certainly a
   misconfiguration (or a bundle that would exceed IBM MQ's message limit anyway). It is
   **not built** — it is logged, alerted, and handed back to the pipeline as a failed /
   poison item, the same as any other bad-data config. This is a data-quality guard, not a
   silent truncation.
2. **Streaming accumulation on the account and alias stages.** Read those result sets with a
   row callback that accumulates into the per-parent maps rather than materialising a full
   `List<Row>` first, and abort with a clear error if a running row total crosses a hard
   heap-safety cap (`commander.read.page-row-hard-limit`, e.g. 1,000,000). Protects the pod
   even if the per-config ceiling is set generously.

This also connects to the still-open **bundle-size bound** noted in
`../../01-requirements/requirements-gaps.md` — the ceiling here is the retrieval-side half of it.

---

## 5. The pure assembler

A single static, dependency-free function:

```
assemble(configs, scopes, paymentTypes, accounts, aliases, recipients) -> List<ResolvedConfig>
```

- Groups each level's flat rows by parent id (`accounts` by assignment id, `assignments` by
  scope id, `scopes` by config id, `recipients` by id) into maps, then builds each config's
  tree top-down with `getOrDefault(..., emptyList())`.
- **Zero-scope configs fall out for free** (requirement 7) — the scope map has no entry for
  that config id → empty `scopes` list → a config-only `ResolvedConfig`.
- Checks the **account/alias mutual-exclusivity invariant** per payment-type assignment (both
  non-empty is illegal — see below).
- No database, no Spring context — unit-tested with hand-built row fixtures.

### A mutual-exclusivity violation is isolated per config, not fatal for the page

`goal.md` §2 says a `PaymentTypeAssignment` carries accounts **or** aliases, never both. If the
assembler sees a payment-type assignment with both sets non-empty, it does **not** throw and
abort the page. It marks *that config* as a bad-data / poison item — logged, alerted, handed
back to the pipeline exactly like the `max-accounts-per-config` ceiling breach (§4) — and
carries on assembling every other config in the page.

This is deliberately the **same choice** the sibling account-ceiling guard makes, for the same
reason: one malformed config should not deny service to the hundreds of well-formed configs
sharing its page, and the pipeline already has a first-class channel for "this config could not
be built" (`FAILED_POISON`, alertable). Failing the whole page would also make the blast radius
of a single bad row depend on which page it landed on — noise, not signal.

The one nuance worth watching operationally: a violation here can mean bad source data **or** a
bug in the staged reads (e.g. an account and an alias query keyed by the wrong parent id and
cross-joining). A *single* config tripping it is almost certainly data; a *high rate* across a
page or run is almost certainly a code bug and should page louder than a lone poison item.
Emit the per-config alert with enough context (config id, the offending assignment id, both
counts) to tell the two apart, and put a run-level "poison rate" threshold on the pipeline
side.

---

## 6. Output shape — `ResolvedConfig`

An **explicit typed record**, not an opaque property map:

```java
public record ResolvedConfig(
    long id,                    // surrogate — the identity/dedup key the pipeline uses
    int configId,               // business key — how on-demand callers address it
    ReportType reportType,
    String reportVersion,
    String reportFrequency,
    AccountFormat accountFormat,
    boolean isActive,
    boolean isBundled,
    boolean isPaginated,
    boolean isEmptyReportAllowed,
    // ...one field per ReportConfig column — the set is closed and known
    Recipient recipient,
    List<ResolvedScope> scopes  // empty (never null) for a config-only result
) {}

public record ResolvedScope(long agreementScopeId, List<ResolvedPaymentType> paymentTypes) {}
public record ResolvedPaymentType(
    long paymentTypeAssignmentId, String paymentType,
    List<ResolvedAccount> accounts, List<ResolvedAlias> aliases   // exactly one non-empty
) {}
```

- **Every `ReportConfig` column is a named field** (requirement 6). BR4 requires the
  downstream message to carry each property *by name*; assembly and message-building reference
  them individually, so a stringly-typed map would push untyped access through the whole
  pipeline. The set of columns is closed (it's a table), and "add a column → add a field + one
  mapper line" is a trivial, localised change. **Decision: typed record.**
- **Both `id` and `configId` are carried** (requirement 8). The pipeline's `WorkItem` /
  `Outbox` `config_id` is the **surrogate `id`** (stable, never reused); `configId` is only
  the on-demand addressing key.
- `RowMapper`s live in one shared class — the single place that knows each projection's column
  set.

---

## 7. PHT — resolve, then a pure balance merge (requirement 9)

The PHT entry point:

1. Resolve the recipient: `Agreement → AgreementVersion (status ACTIVE) → AgreementScope`
   matched to the external engagement identifier and the PHT report type → one
   `MessageRecipientId`.
2. `resolvePage` for that recipient's single active PHT-report-type config — the ordinary
   staged walk, one config.
3. **Merge the pushed balances** — a pure function, not a query:

   ```
   mergeBalances(ResolvedConfig, Map<AccountKey, Balance>) -> ResolvedConfig
       AccountKey = (clearingNumber, accountNumber)
   ```

   Each resolved account is looked up in the balance map: **matched** → the account carries
   the pushed balance/settlement amount; **unmatched** → the account is dropped from the
   result (a PHT push only ever covers the accounts it carries). Balances with no matching
   resolved account are ignored. Aliases are untouched.

This keeps the DB-read part identical to the other entry points; the merge takes non-DB input
and is independently testable.

---

## 8. The one schema addition

A single **general-purpose user-defined table type** — e.g.
`CREATE TYPE dbo.BigIntIdList AS TABLE (Id BIGINT NOT NULL PRIMARY KEY)` — reused at every
TVP-backed level and by `reResolve`. Not report-specific; one type, one migration.

**This is the only new object this layer needs in the `CAMT` schema.** Whether the schema may
be extended at all (`goal.md` §6 flags this) is the gating question — confirm it before build.

**Fallback if a new type is genuinely disallowed:** chunk id sets over ~2,000 into sub-batches
and issue multiple queries per level, merging in the adapter. This trades a fixed 6 round
trips/page for a *variable* count that spikes exactly on wide pages — working against
requirement 1's flat-cost goal. Take it only if the TVP type is off the table.

---

## 9. Hexagonal placement

| Piece | Layer |
|---|---|
| `ReportConfigResolutionPort` (`resolvePage`, `reResolve`), `RecipientPort`, `PhtRecipientPort` | application (ports) |
| JDBC adapter: the staged SQL, keyset paging, TVP binding, `RowMapper`s | adapter |
| `assemble(...)`, `mergeBalances(...)`, the `Resolved*` records, the `scope_key` grammar | domain (pure) |

The adapter depends on the SQL Server JDBC driver only for TVP binding
(`SQLServerDataTable` + `setStructured`); everything else is plain `JdbcTemplate`.

---

## 10. Configuration

| Property | Default | Purpose |
|---|---|---|
| `commander.read.page-size` | 500 | keyset page size — memory (one page's resolved trees held at once) vs. round-trip count vs. recovery granularity |
| `commander.read.tvp-query-timeout-seconds` | 15 | timeout on the fan-out (TVP) stages |
| `commander.read.in-list-guard` | 2000 | fail-fast ceiling for the plain-`IN` queries — the Level 1→2 scope query and the recovery Level-1 (`Id IN (:ids)`) query |
| `commander.read.max-accounts-per-config` | 25000 | a config resolving to more accounts than this → treated as bad data, handed to the pipeline as a failed/poison item, not built (§4, *Bounding page width*) |
| `commander.read.page-row-hard-limit` | 1000000 | hard heap-safety cap on total account/alias rows accumulated for one page — abort with a clear error if crossed |

The read layer uses its **own** `JdbcTemplate` (its own timeout), not a bean shared with other
repositories.

---

## 11. Why not the alternatives

The chosen design is **Option A** from `archive/solution_v01.md`. The other two aren't wrong — they'd
work — but each trades something this layer specifically needs for something it doesn't.

### Resolving per config (the anti-pattern)

Not a real option, but worth naming as the baseline the rest improve on: a query set per
`ReportConfig` → ~5 round trips × 10,000 configs ≈ **50,000 round trips per run**, plus 10,000
tiny query plans and 10,000 result-set setups. This is exactly what requirement 1 forbids and
what the staged design replaces with ~60 round trips.

### Option B — single stored procedure, multiple result sets

*Mechanism:* one `usp_ResolveConfigPage(@ids dbo.BigIntIdList READONLY)` walks the hierarchy
server-side, holding the intermediate id sets (scope ids, assignment ids …) in local temp
tables, and returns the same six flat result sets in one call — read via a `CallableStatement`
and a `getMoreResults()` loop.

*What it buys:* the fewest round trips — one per page instead of six. For a whole run that's
~50–100 fewer round trips, on the order of **0.1–0.3 s of latency**.

*Why it's not the fit:*

1. **The saving is immaterial and it's the wrong kind of saving.** The six `SELECT`s execute
   byte-for-byte the same work whether issued from the app or from inside a proc — B changes
   *where the calls originate*, not the DB's query cost or the wire volume. Round-trip *count
   per config* — the thing that actually scales badly — is already solved by batching; B only
   shaves fixed per-page latency you won't notice.
2. **A stored procedure is application logic living in the database.** The TVP type is one
   general-purpose line. A proc is a versioned object with its own migration, a DBA review
   gate, redeploy coupling (a schema change to tweak a `WHERE` clause), and a tendency to
   accrete special-cases. "How resolution works" is now split across two repos and two
   languages.
3. **`reResolve` doesn't fit its shape.** The proc is built around "start from config ids,
   walk down". Recovery's presence-check wants the same walk but with per-config-batch
   absence/failure semantics. Serving it means a second procedure or a mode flag branching in
   T-SQL — duplicated logic in the language that's hardest to unit-test.
4. **Testability inverts.** With Option A each level is a small repository method covered by
   fast tests, and the assembler is pure. B's six-way T-SQL walk can realistically only be
   exercised by an integration test against a real SQL Server — slower CI, and edge cases
   (an empty level, the account/alias mutual-exclusivity violation, one level erroring
   mid-walk) are harder to force.
5. **Partial-failure visibility.** If the accounts `SELECT` errors, JDBC sees one failed call
   with no structure. Option A sees "the accounts stage failed for this page", other stages'
   results intact, the tri-state cleanly attributable. B would need the proc itself to
   `TRY/CATCH` per level and emit a status column — more T-SQL, now doing error-reporting
   design.

*When B would win:* if profiling a real deployment shows round-trip **latency** (a
high-latency link to the DB, say) is a measured bottleneck — an evidence-backed reason to
accept a proc's operational cost, not a default assumption.

### Option C — one flattened `LEFT JOIN` per page

*Mechanism:* a single wide `SELECT` per page —
`ReportConfig LEFT JOIN ReportAgreementScope LEFT JOIN AgreementScope LEFT JOIN
PaymentTypeAssignment LEFT JOIN (AccountAssignment / AliasAssignment) LEFT JOIN Recipient`,
filtered by the page's config ids via the TVP. One denormalised rowset: one row per
`(config × scope × payment-type × account-or-alias)` leaf. The assembler groups by config id
and de-duplicates the repeated parent columns.

*What it buys:* a true single round trip per page, and — unlike B — no stored procedure, just
plain SQL.

*Why it's not the fit:*

1. **Row multiplication, then column repetition on top.** A config with 2 scopes × 3 payment
   types × 20 accounts is 120 rows for 20 accounts of real data — a 6× inflation before you
   count columns. And *every* one of those 120 rows carries a full copy of the config's ~20
   property columns and the recipient's 4. Option A fetches each config row and each recipient
   row exactly **once**. For a page holding a few wide bundled configs, most bytes transferred
   and most row-mapping CPU is redundant copies — a "hot page" blow-up that is *worse* than
   Option A's hot page ("many narrow account rows"), because C's rows are wide.
2. **Single failure domain — the tri-state degrades.** One query returns everything or throws.
   You can still read "id absent from a successful result" as `CONFIRMED_ABSENT`, but you lose
   per-level isolation: a transient timeout on the account join fails the whole page's
   resolution, including the levels that would have succeeded. Option A's independent stages
   let a transient failure be scoped and retried narrowly.
3. **`reResolve` still needs a second query shape** — a differently-filtered join for the
   leaf-keyed presence check — so C doesn't even deliver "one query shape" once recovery is in
   scope. Same duplication as B, without B's round-trip edge to offset it.
4. **Optimiser unpredictability.** A five-way `LEFT JOIN` with variable cardinality at every
   level (0..N scopes, 1..N payment types, 0..N accounts) is among the hardest shapes for the
   SQL Server optimiser to plan stably — small statistics drift can flip join order or
   strategy and swing runtime by an order of magnitude. Option A's queries are each a targeted
   seek with a predictable plan a DBA can index and reason about individually.

*When C would be acceptable:* only if adding **any** new schema object — even the one-line TVP
type — is genuinely forbidden. Then C with chunked `IN` instead of the TVP is the
zero-schema-change option, at the cost of everything above. The minimalist fallback, not a
contender.

---

## 12. Open items

- **Verify the required indexes (§4) exist** on the `CAMT` schema — the composite
  `ReportConfig (ReportType, ReportFrequency, IsActive, Id)` in particular. Missing ones are a
  migration prerequisite.
- **Confirm `dbo.BigIntIdList` (§8) is an acceptable `CAMT`-schema addition.** Everything here
  assumes it; the chunked-`IN` fallback is the only alternative.
- **Sign off the `scope_key` grammar (§3) jointly with the pipeline** — it also feeds
  `UQ_WorkItem_Identity` and `Outbox.scope_key` in `solution_v08.md`.
- **`max-accounts-per-config` / `page-row-hard-limit` values (§4, §10)** — set from the real
  distribution of accounts per bundled config; ties to the open bundle-size bound in
  `../../01-requirements/requirements-gaps.md`.
- **Page-size tuning** once real config→scope→leaf ratios are measured (the "500 → thousands
  of scope ids" figure is illustrative).
- **Does recovery ever need refreshed recipient data**, or only leaf (account/alias) data?
  If only leaves, `reResolve` can skip step 6.
