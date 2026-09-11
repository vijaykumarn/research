# Commander — Data Retrieval: Problem Brief

I want a fresh design for the **data-retrieval layer** — the step that, given a set of report
configurations to process, fetches everything needed to build their report messages. Please
propose **2–3 distinct approaches**, with trade-offs. This brief states the problem and the
constraints; it does not prescribe a design.

This is a sibling concern to the message-production pipeline (`../message-pipeline/`) and
scheduling (`../scheduling/`). Those own *when* work runs, *tracking* it, and *publishing*
results. This layer owns only: **turn a set of `ReportConfig`s into fully-resolved input for
message assembly.**

---

## 1. What it produces

For each `ReportConfig` in the input set, a fully-resolved structure containing:

- **The config's own properties** — *all of them* (active flag, bundling flag, account format,
  pagination flag, report type, report version, frequency, start/end dates, business key,
  surrogate id, …). The downstream message must carry every property, whether or not it
  affects processing.
- **The resolved scope hierarchy** under that config (see §2), if the config has one.
- **The recipient** the message is addressed to.

The message-assembly step (in the pipeline) takes this and produces one or more messages per
config according to the bundling rule (bundled → one per payment type; unbundled → one per
account/alias; no scope → one "config-only" message). Assembly is **out of scope here** — this
layer stops at "resolved structure per config".

---

## 2. The hierarchy to resolve

The data lives in the existing SQL Server `CAMT` schema, across these tables:

```
ReportConfig
  └─ (via ReportAgreementScope) AgreementScope        — 0..N per config
        └─ PaymentTypeAssignment                       — 1..N per scope
              ├─ AccountAssignment                     — 0..N per assignment
              └─ AliasAssignment                       — 0..N per assignment

ReportConfig.MessageRecipientId ──▶ Recipient          — exactly 1 per config

(PHT path only)
Agreement ─▶ AgreementVersion (status ACTIVE) ─▶ AgreementScope
   used to resolve a Recipient from an external engagement identifier
```

- A config with **zero** linked `AgreementScope`s is valid — it resolves to just its own
  properties + recipient ("config-only").
- `ReportConfig` has both a surrogate `Id` and a business `ConfigId`.
- `PaymentTypeAssignment` carries an `AccountAssignment` set **or** an `AliasAssignment` set
  (mutually exclusive per assignment).

---

## 3. Three entry points, one shared resolution

The layer is invoked from three places. They differ only in **how the config set is chosen** —
the resolution of scopes/accounts/aliases/recipient should be the same work afterward.

| Caller | How the config set is selected | Typical size |
|---|---|---|
| **Scheduled** | A page of `ReportConfig` where `report_type = ?` and `frequency = ?` and `is_active = 1`, walked page after page | 1,000 – 10,000+ configs per run, in pages |
| **On-demand** | A caller-supplied list of **`ConfigId`s** (the business key), from an inbound message | small, bounded (tens) |
| **PHT** | Resolve one `Recipient` from an external engagement identifier via `Agreement → AgreementVersion(ACTIVE) → AgreementScope`, then that recipient's single active config for the PHT report type. The inbound message also carries **account balances** that must be merged into the resolved accounts (matched by clearing + account number); accounts the push does not cover are dropped from that message. | 1 config |

---

## 4. Volume and environment

- A single scheduled run resolves **1,000 to 10,000+ configs**. Resolving one config at a time
  (a query set per config) is **not viable** — tens of thousands of round-trips per run.
- Runs are processed **page by page** (the pipeline creates its tracking rows per page,
  interleaved with resolution). A page is on the order of a few hundred configs; the exact
  size is a tunable, not fixed.
- The layer is **read-only and stateless**. Multiple pods run concurrently, but this layer
  needs no coordination — the pipeline owns run tracking, checkpoints, and recovery.
- **Read consistency:** "the data as of when this run reads it" is acceptable. A config or
  scope change made mid-run need not be reflected in that run — it is picked up on the next
  run. No snapshot isolation is required.

---

## 5. Requirements the approach must satisfy

1. **Resolution cost is roughly per-page, not per-config.** No query fan-out that scales with
   the number of configs in a page.
2. **One shared resolution core** for all three entry points; only the config-set selection
   differs.
3. **Pagination for the scheduled path** that is stable while `ReportConfig` rows are being
   inserted/updated concurrently, and does not get more expensive as the run progresses
   (no growing-offset penalty).
4. **Recipient resolution is batched with the page**, not done per config.
5. **Flat query results are assembled into per-config structures by a pure, database-independent
   step** — no DB access in the assembly logic, so it is unit-testable in isolation.
6. **Every `ReportConfig` property reaches the output**, including ones that don't affect
   resolution or assembly logic.
7. **Zero-scope configs** resolve successfully to a config-only structure.
8. **Both `Id` and `ConfigId` are carried through** to the output — the scheduled path pages by
   the surrogate `Id`; the on-demand path addresses configs by `ConfigId`; downstream identity
   keys must be able to use whichever they need.
9. **PHT balance merge** — the resolved accounts for the PHT config are matched against the
   inbound message's balances by clearing + account number; matched accounts carry the pushed
   balance, unmatched accounts are excluded from that message.
10. **A tri-state resolve result for recovery re-resolution.** When the pipeline's recovery
    re-resolves a page it needs to distinguish, per already-known scope key:
    *found* / *confirmed absent* (the account/scope was genuinely removed → the pipeline
    retires that work item) / *query failed* (transient — retry, don't retire).
11. **SQL Server parameter limits.** A page-sized set of ids expanded through the hierarchy can
    exceed SQL Server's per-statement parameter cap. The approach must handle that (and state
    whether it needs any new database object to do so — the `CAMT` schema is pre-existing and
    whether it may be extended is itself a question to surface).

---

## 6. Fixed constraints

- **Spring Boot** service, **SQL Server**, the existing **`CAMT` schema** (tables in §2).
- Hexagonal layering is expected: repository **ports** in the application layer, a JDBC
  **adapter** implementing them, the tree/assembly logic as **pure domain** code.
- No ORM is assumed; plain JDBC / `JdbcTemplate`-style access is expected.

---

## 7. Out of scope

- Building the report messages (assembly, bundling, fan-out) — that's the pipeline.
- Run tracking, checkpoints, the outbox, recovery orchestration — the pipeline.
- Deciding *which* configs are due and *when* — scheduling.
- Any caching layer (may be discussed as a trade-off, but the baseline is "read fresh each
  run").
