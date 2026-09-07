# CAMT Report Message Service — Redesign Brief

I want to redesign this service from a fresh mind. Please propose **2–3 distinct architecture
options** for how to structure it, with the trade-offs of each. This brief is deliberately
lean — it states what the service must do and the fixed technology constraints, and nothing
more.

---

## Purpose

A backend service that **generates CAMT report messages and publishes them to IBM MQ**. It
does not render the final reports — a downstream "Executor" app consumes the messages off MQ
and does that. This service decides which reports are due, for whom, and over what time
window; resolves the account / alias data; packages each as a JSON `ReportMessage`; and puts
it on the report type's queue.

## Report types

Six, in three pairs:

| Report type | Category |
|---|---|
| `CAMT052B`, `CAMT052BT` | Intraday account report |
| `CAMT053S`, `CAMT053E` | End-of-day statement |
| `CAMT054C`, `CAMT054D` | Debit / credit notification |

## Inbound triggers

| Trigger | Source | What it does |
|---|---|---|
| **Scheduled** | A scheduled job fires for a `(reportType, frequency)` pair | Pages through all active `ReportConfig` rows matching that pair and produces a message for each |
| **On-demand** | JMS listener on `CAMT.ONDEMAND.QUEUE` | Consumes a message carrying a caller-supplied list of config identifiers and produces a message for each. Queue-driven, not a REST API. |
| **PHT** (external balance push) | JMS listener on `CAMT.PHT.QUEUE` | Parses a semicolon-delimited, fixed-width body containing account balances; resolves the recipient from an external engagement identifier via the agreement chain; produces a `CAMT052B` message that carries the pushed balances |

All three converge on the same data-resolution, message-assembly, and publish path.

## Outbound

One IBM MQ queue per report type. The message body is JSON (`ReportMessage`). A downstream
"Executor" service is the sole consumer.

## Fixed constraints

- **Spring Boot** service.
- **IBM MQ** for messaging (JMS).
- **SQL Server** for persistence.
- Scheduling via **Quartz**.

## Deployment & resilience context

- The service runs as **multiple concurrent pods** on a container platform. All pods are
  identical, share the one SQL Server database and the one MQ manager, and any of them may be
  running any of the three trigger paths at any moment.
- **Any pod can be terminated at any time** — rolling deployment, node eviction, resource
  limit, or crash — with no clean shutdown guaranteed, including mid-run.
- A scheduled run can span a large number of `ReportConfig` rows (into the thousands), so an
  interrupted run represents meaningful lost work if it has to start over.
- The architecture options should say how they handle: only one pod acting on a given
  scheduled trigger / unit of work at a time; detecting and continuing (or safely redoing)
  work left incomplete by a terminated pod; and not publishing a duplicate message to MQ when
  work is retried or resumed.

## In scope / out of scope

- **In:** the three inbound triggers, resolving the report data, assembling the
  `ReportMessage`, publishing it to the correct queue.
- **Out:** rendering the actual report documents — the downstream Executor does that.
