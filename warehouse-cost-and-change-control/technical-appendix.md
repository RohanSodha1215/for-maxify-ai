# Technical appendix: a cost and change-control tool for a BigQuery data warehouse

*Also available as a [PDF](warehouse-cost-and-change-control-technical-appendix.pdf). The two-page case study is [here](README.md).*

**Role:** founding and main developer · **Period:** April 2025 to present · **Status:** in production

**Stack:** Python (Flask, SQLAlchemy), PostgreSQL, Vue 3, BigQuery, Google Cloud Storage, Docker

This appendix accompanies the two-page case study of the same name. It repeats the summary and then describes each part in detail.

## The problem

Our company keeps its analytics data in BigQuery, a cloud data warehouse that charges for the data stored and for the compute each query uses. Hundreds of people create tables and write queries. Nobody removes a table when it stops being used, and nobody acts on the provider's suggestions for laying tables out more cheaply, so the bill only grows.

The data engineering team was also the bottleneck for routine changes. Adding a column or creating a table meant asking an engineer in chat, who then made the change by hand with no record of who approved it.

## What I built

An internal web tool that finds waste in the warehouse, removes it safely, and puts routine schema changes behind an approval step. I started it on an empty project skeleton and built the parts described here. Query optimisation, also part of this tool, has [its own case study](../llm-query-optimisation/README.md); the tool's remaining features were built by teammates and are left out.

This document covers four parts of mine. The first, the cleanup of inactive tables, is the largest:

```
Nightly scan ── merges the last few days of query logs into a
     │          "last access per table" state table, then reads it
     ▼
 Detected ───── no query has read the table for N days
     │          (an owner can exclude it at any time)
     ▼  wait 1
 Pending ────── table is renamed, so anything still using it
     │          fails loudly while the data is intact
     ▼  wait 2  (a person can restore it; an admin can pause this step)
 Archived ───── exported to cloud storage with a retention period
     ▼
 Deleted
```

Every stage, waiting period and exclusion pattern is a setting in the UI. An overview page shows the storage cost of what was removed, priced from each table's actual billing model.

**Example.** A team stops using a large table. After 180 days with no query reading it, the scan lists it as detected. After the first wait the tool checks, live, that nothing has written to the table recently, then renames it. Nobody complains during the second wait, so the tool exports the table to cloud storage and deletes it. Had a job still depended on it, the job would have failed on the rename and the owner could have restored the table with one click.

The other three parts follow. Under all of them sits a runtime I also built: a metadata store, a background job executor and cron scheduler (first versions), heartbeat-based leader election so scheduled work runs once across replicas, configuration editable in the UI, Google sign-in with editor and admin roles, and the container build.

### Schema change requests

A user asks for new columns on a table, or for a new table, in a form. An admin sees the exact difference against the live table and approves or declines. A background worker applies approved requests.

- **Two request types with different payloads.** An add-column request stores only the new columns. At apply time the worker reads the live schema, appends the new columns and writes the result. Because a request never carries a full schema, it cannot drop or change an existing column, and columns that someone else added in the meantime are kept. A create-table request stores the full table definition.
- **Validation in two places that must agree.** The form and the server apply the same rules for types, names, nested fields, partitioning and clustering. A shared test fixture runs against both, so the UI cannot enable a button that the server will refuse.
- **A closed state machine.** There are six statuses, and the allowed transitions are listed in one table. Every status change is a compare-and-set in the database, checked against that table, and is written to an append-only event log in the same transaction. At most one open request per table is allowed, enforced by a unique index.
- **Approval is a decision about the live table.** On approve, the server reads the table again and refuses if the request no longer applies. The approval and the background job are written in one transaction, so an approved request without a job cannot exist.
- **One writer at a time.** The worker claims a request with a lease that expires. Before writing to BigQuery it confirms that it still holds the lease. The write itself is conditional on the table version it read, with one re-read and retry if that version changed.
- **Crash recovery without a sweeper.** If the worker dies after its write landed but before the database was updated, the next attempt sees that the columns are already there and identical, and completes the request without writing. For a new table it does this only on a second or later attempt and only if the existing table matches the approved definition; otherwise the table is someone else's and the request goes back for review.
- **No "failed" status.** A change that cannot be applied returns to the approval queue. The admin sees the error, classified as temporary (approve again), a permission problem (fix it, then approve) or a conflict (decline). The requester sees only the outcome and, on a decline, the reason.

### Clustering suggestions

Clustering sorts a table's storage by chosen columns, so a query that filters on those columns reads less data. The cloud provider publishes recommendations for which tables to cluster and on which columns, with an estimate of the compute saved, but nobody was acting on them.

- A scheduled job pulls the recommendations for the configured projects, drops those already in effect by comparing them with each table's live settings, and replaces the stored list in one transaction.
- A page lists each suggestion with its estimated monthly saving. A user applies it, which runs as a background job, or dismisses it. A dismissed suggestion stays dismissed across refreshes, because it is keyed by table.
- Applied suggestions are written to a log.

This is the simplest part of the tool. Its value was turning a list nobody read into a one-click action. The saving shown per suggestion is the provider's estimate; the tool does not measure it afterwards.

### API for automated agents

An internal chat bot files schema-change and access requests for users. I added the API it uses, with one rule: an agent can file and read requests, but can never decide one.

- An agent call carries a shared token and a header naming the person it acts for. The token is compared in constant time; if no token is configured, the call is refused instead of matching an empty value.
- The named person is recorded as the requester. The audit log records the caller as "internal API" and the person as claimed, not proven, because a shared token cannot prove which person asked. The body of the request cannot override the header.
- Every approve, decline and configuration route refuses any call that carries the token, and it does so before the role check. The role check alone is not a safe gate here: it grants admin to everyone when login is disabled or before the first admin exists.
- For schema changes, a client-supplied request id makes a retried submission return the original request instead of creating a second one.

## Design decisions

- **A destructive step fails closed.** If the access data cannot be refreshed, the whole nightly run stops. If a table's last-modified time cannot be read live at decision time, the table is held back. A cached value can be stale, and a wrongly kept table costs far less than a wrongly deleted one.
- **Rename before delete.** A rename is cheap to undo and surfaces hidden users of a table better than any log analysis.
- **Derive state instead of storing it.** The deletion date is computed from the day the table entered the pending stage plus the current setting, so changing the setting takes effect at once. The log-ingest position is the newest timestamp in the state table itself, so first run, catch-up and rebuild are one code path.
- **Schema changes can only add.** No request can remove data, so the worst outcome of a wrong approval is an unused column.
- **Automated agents can ask; only people decide.**

## Results

- Inactive table cleanup has saved about USD 1,000 a month.
- Clustering suggestions have saved about USD 1,000 a month.
- Moving the inactive-table scan from the full query-log history to the incremental state table cut its nightly log read from about 30 GB per scanned project to about 0.3 GB in total, measured on production data. The cost no longer grows with the number of projects scanned.
- Column additions and table creations go through a request, an approval and an audit trail instead of a chat message and a manual change.

## What I personally built

- The service's foundation: metadata store, the first background job executor and scheduler, leader election, configuration, sign-in and roles, deployment packaging.
- Inactive table cleanup, end to end: scan, staged deletion, exclusions, archival, cost reporting, the incremental last-access table, and the UI.
- Schema change requests, end to end: validation on both sides, the state machine and event log, the lease-based worker with crash recovery, and the UI.
- Clustering suggestions: the scheduled refresh, the apply and dismiss actions, and the UI.
- The token-based API that lets automated agents file and read requests but not decide them.

## Why the code is not shared

The repository belongs to my employer and is private. I am happy to walk through the design and the code in a live session.
