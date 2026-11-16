# Representative work

Most of my professional work is in private repositories that belong to my employer, so I cannot share the code. This repository holds anonymized case studies of that work instead.

Each case study explains the problem, what I built, how it works, the main design decisions, and which parts were mine. Internal names and data are left out.

## Case studies

| Case study | What it is | Read |
| --- | --- | --- |
| [On-call assistant bot](#on-call-assistant-bot-for-a-data-engineering-team) | A Slack bot that answers, prepares or escalates the requests a data engineering team's on-call engineer receives. | [case study](on-call-assistant-bot/README.md) · [PDF](on-call-assistant-bot/on-call-assistant-bot-case-study.pdf) |
| [Warehouse cost and change-control tool](#cost-and-change-control-tool-for-a-bigquery-data-warehouse) | An internal web tool that safely removes unused tables, applies the provider's clustering suggestions and puts schema changes behind an approval step. | [case study](warehouse-cost-and-change-control/README.md) · [PDF](warehouse-cost-and-change-control/warehouse-cost-and-change-control-case-study.pdf) · [appendix](warehouse-cost-and-change-control/technical-appendix.md) |
| [Verified LLM query rewrites](#verified-llm-rewrites-of-expensive-warehouse-queries) | A pipeline that has a language model rewrite expensive queries and accepts a rewrite only after a measured run proves it returns the same rows for less compute. | [case study](llm-query-optimisation/README.md) · [PDF](llm-query-optimisation/llm-query-optimisation-case-study.pdf) |

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

**Read:** [case study](warehouse-cost-and-change-control/README.md) · [PDF, 2 pages](warehouse-cost-and-change-control/warehouse-cost-and-change-control-case-study.pdf) · [technical appendix](warehouse-cost-and-change-control/technical-appendix.md) · [appendix PDF, 3 pages](warehouse-cost-and-change-control/warehouse-cost-and-change-control-technical-appendix.pdf)

**Role:** founding and main developer · **Period:** April 2025 to present · **Status:** in production

**Stack:** Python (Flask, SQLAlchemy), PostgreSQL, Vue 3, BigQuery, Google Cloud Storage, Docker

An internal web tool for a company whose data warehouse bill only grew: nobody removed unused tables or acted on the provider's suggestions for cheaper table layouts, and schema changes were made by hand with no record of approval. The tool finds inactive tables and removes them in reversible stages, applies clustering suggestions with one click, and puts column and table changes behind an approval step. Table cleanup and clustering have each saved about USD 1,000 a month.

What I personally built:

- The service's foundation: metadata store, the first background job executor and scheduler, leader election, configuration, sign-in and roles.
- Inactive table cleanup, end to end: scan, staged and reversible deletion, archival, cost reporting.
- Schema change requests, end to end: validation, the state machine and event log, the lease-based worker with crash recovery.
- Clustering suggestions, and the API that lets automated agents file and read requests but not decide them.

Query optimisation is also part of this tool and has [its own case study](#verified-llm-rewrites-of-expensive-warehouse-queries). The tool's other features were built by teammates; the case study covers only my parts.

### Verified LLM rewrites of expensive warehouse queries

**Read:** [case study](llm-query-optimisation/README.md) · [PDF, 4 pages](llm-query-optimisation/llm-query-optimisation-case-study.pdf)

**Role:** designer and sole developer · **Period:** May 2026 to present · **Status:** in production

**Stack:** Python (Flask), PostgreSQL, Vue 3, BigQuery Python SDK, LLM APIs via an OpenAI-compatible proxy

A pipeline that has a language model propose a cheaper version of an expensive warehouse query, and accepts the proposal only after the warehouse itself has shown that it is valid, returns the same number of rows and uses less compute. Both queries run on the same snapshot of the data with the cache off, and a losing rewrite is cancelled the moment it costs more than the original. Owning teams receive the verified rewrites with before-and-after numbers; the rewrites they have applied save about USD 4,000 a month.

What I personally built (all of it):

- The pipeline: context collection, the correction loop that feeds the warehouse's error back to the model, the time-travel rewrite, and the measured comparison with early cancellation, timeouts and abort.
- A mode that starts from a past production job and reuses its recorded statistics, so only the rewrite is executed.
- A model client with two providers, strict structured output, settings that apply without a restart, and clear reporting of cut-off replies.
- Parallel runs, a command-line version, the hand-over messages to owning teams, and the web pages.

It is part of the warehouse tool above and starts past-job runs from that tool's job history, which is a separate feature.

## Code walkthrough

The repositories are private, so the code is not here. I am happy to walk through the design and the code in a live session.
