# Case study: a cost and change-control tool for a BigQuery data warehouse

*Also available as a [PDF](warehouse-cost-and-change-control-case-study.pdf). A longer [technical appendix](technical-appendix.md) describes each part in detail ([PDF](warehouse-cost-and-change-control-technical-appendix.pdf)).*

**Role:** founding and main developer · **Status:** in production

**Stack:** Python (Flask, SQLAlchemy), PostgreSQL, Vue 3, BigQuery, Google Cloud Storage, Docker

## The problem

Our company keeps its analytics data in BigQuery, a cloud data warehouse that charges for the data stored and for the compute each query uses. Hundreds of people create tables and write queries. Nobody removes a table when it stops being used, and nobody acts on the provider's suggestions for laying tables out more cheaply, so the bill only grows.

The data engineering team was also the bottleneck for routine changes. Adding a column or creating a table meant asking an engineer in chat, who then made the change by hand with no record of who approved it.

## What I built

An internal web tool that finds waste in the warehouse, removes it safely, and puts routine schema changes behind an approval step. I started it on an empty project skeleton and built the parts described here. Query optimisation, also part of this tool, has [its own case study](../llm-query-optimisation/README.md); the tool's remaining features were built by teammates and are left out.

The cleanup of inactive tables is the largest part:

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

**The four parts covered here**

- **Inactive table cleanup.** The flow above. Every stage, waiting period and exclusion pattern is a setting in the UI.
- **Schema change requests.** A user requests new columns or a new table in a form. An admin sees the exact difference against the live table and approves or declines. A background worker applies approved changes, and every step is written to an append-only event log.
- **Clustering suggestions.** A page lists the cloud provider's recommendations for sorting a table's storage so that queries read less, and applies one with a click.
- **An API for automated agents.** An internal chat bot can file and read requests on a user's behalf, but cannot approve them.

Under these sits a runtime I also built: a metadata store, a background job executor and scheduler (first versions), leader election so scheduled work runs once across replicas, configuration editable in the UI, and Google sign-in with roles.

**Example.** A team stops using a large table. After 180 days with no query reading it, the scan lists it as detected. After the first wait the tool checks, live, that nothing has written to the table recently, then renames it. Nobody complains during the second wait, so the tool exports the table to cloud storage and deletes it. Had a job still depended on it, the job would have failed on the rename and the owner could have restored the table with one click.

A separate technical appendix describes each part in more detail.

## Design decisions

- **A destructive step fails closed.** If the access data cannot be refreshed, the whole nightly run stops. If a table's last-modified time cannot be read live at decision time, the table is held back. A wrongly kept table costs far less than a wrongly deleted one.
- **Rename before delete.** A rename is cheap to undo and surfaces hidden users of a table better than any log analysis.
- **Derive state instead of storing it.** The deletion date is computed from the day the table entered the pending stage plus the current setting, so changing the setting takes effect at once.
- **Schema changes can only add.** A request carries only the new columns, never a full schema, so no request can drop a column. If the worker crashes after its write landed, re-approving completes as a no-op.
- **Automated agents can ask; only people decide.** Any call carrying the agent token is refused on approval routes before the role check runs.

## Results

- Inactive table cleanup has saved about USD 1,000 a month.
- Clustering suggestions have saved about USD 1,000 a month.
- Moving the inactive-table scan to an incremental state table cut its nightly log read from about 30 GB per scanned project to about 0.3 GB in total, measured on production data.
- Column additions and table creations go through a request, an approval and an audit trail instead of a chat message and a manual change.

## What I personally built

- The service's foundation: metadata store, the first background job executor and scheduler, leader election, configuration, sign-in and roles, deployment packaging.
- Inactive table cleanup, end to end: scan, staged deletion, exclusions, archival, cost reporting, the incremental last-access table, and the UI.
- Schema change requests, end to end: validation, the state machine and event log, the lease-based worker with crash recovery, and the UI.
- Clustering suggestions: the scheduled refresh, the apply and dismiss actions, and the UI.
- The token-based API that lets automated agents file and read requests but not decide them.

## Why the code is not shared

The repository belongs to my employer and is private. I am happy to walk through the design and the code in a live session.
