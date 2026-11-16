# Representative work

Most of my professional work is in private repositories that belong to my employer, so I cannot share the code. This repository holds anonymized case studies of that work instead.

Each case study explains the problem, what I built, how it works, the main design decisions, and which parts were mine. Internal names and data are left out.

## Case studies

| Case study | What it is | Read |
| --- | --- | --- |
| [On-call assistant bot](#on-call-assistant-bot-for-a-data-engineering-team) | A Slack bot that answers, prepares or escalates the requests a data engineering team's on-call engineer receives. | [case study](on-call-assistant-bot/README.md) · [PDF](on-call-assistant-bot/on-call-assistant-bot-case-study.pdf) |
| [Warehouse cost and change-control tool](#cost-and-change-control-tool-for-a-bigquery-data-warehouse) | An internal web tool that removes unused tables, verifies cheaper query rewrites and puts schema changes behind an approval step. | [case study](warehouse-cost-and-change-control/README.md) · [PDF](warehouse-cost-and-change-control/warehouse-cost-and-change-control-case-study.pdf) · [appendix](warehouse-cost-and-change-control/technical-appendix.md) |

### On-call assistant bot for a data engineering team

**Read:** [case study](on-call-assistant-bot/README.md) · [PDF, 2 pages](on-call-assistant-bot/on-call-assistant-bot-case-study.pdf)

**Role:** designer and main developer · **Period:** June 2026 to present · **Status:** in production since October 2026

**Stack:** Python, an LLM agent framework (steps defined in YAML), Slack, MCP tool servers, sandboxed scripts

A Slack bot for a team whose on-call engineer received about 50 requests a month from other teams, most of them repeats. The bot reads each request and does one of three things: answers it from curated knowledge, prepares a standard request and sends it only after the person confirms, or hands over to the on-call engineer.

What I personally built:

- The bot's overall design, from its first version to the production rollout.
- The triage, answer, request-preparation and escalation flows.
- The design of the alert-investigation and job-failure flows, which a teammate implemented.
- An API on an internal access-request service so that automated agents can file and read requests but never approve them.
- The knowledge base (FAQ entries and request templates) and the request analysis that decided what to build.

The bot runs on two internal platforms built by colleagues: an agent framework and a knowledge registry. I did not build those; the case study explains both.

### Cost and change-control tool for a BigQuery data warehouse

**Read:** [case study](warehouse-cost-and-change-control/README.md) · [PDF, 2 pages](warehouse-cost-and-change-control/warehouse-cost-and-change-control-case-study.pdf) · [technical appendix](warehouse-cost-and-change-control/technical-appendix.md) · [appendix PDF, 4 pages](warehouse-cost-and-change-control/warehouse-cost-and-change-control-technical-appendix.pdf)

**Role:** founding and main developer · **Period:** April 2025 to present · **Status:** in production

**Stack:** Python (Flask, SQLAlchemy), PostgreSQL, Vue 3, BigQuery, Google Cloud Storage, LLM APIs, Docker

An internal web tool for a company whose data warehouse bill only grew: nobody removed unused tables or revisited expensive queries, and schema changes were made by hand with no record of approval. The tool finds and safely removes inactive tables, has a language model propose cheaper queries and accepts one only after a side-by-side run proves it, and puts column and table changes behind an approval step. Query optimisation has saved about USD 4,000 a month, and table cleanup and clustering about USD 1,000 a month each.

What I personally built:

- The service's foundation: metadata store, the first background job executor and scheduler, leader election, configuration, sign-in and roles.
- Inactive table cleanup, end to end: scan, staged and reversible deletion, archival, cost reporting.
- Query optimisation, end to end: the model client, the retry loop, the snapshot-pinned comparison with early cancel.
- Schema change requests, end to end: validation, the state machine and event log, the lease-based worker with crash recovery.
- Clustering suggestions, and the API that lets automated agents file and read requests but not decide them.

The tool also has other features built by teammates; the case study covers only my parts.

## Code walkthrough

The repositories are private, so the code is not here. I am happy to walk through the design and the code in a live session.
