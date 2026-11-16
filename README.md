# Representative work

Most of my professional work is in private repositories that belong to my employer, so I cannot share the code. This repository holds anonymized case studies of that work instead.

Each case study explains the problem, what I built, how it works, the main design decisions, and which parts were mine. Internal names and data are left out.

## Case studies

| Case study | What it is | Read |
| --- | --- | --- |
| [On-call assistant bot](#on-call-assistant-bot-for-a-data-engineering-team) | A Slack bot that answers, prepares or escalates the requests a data engineering team's on-call engineer receives. | [case study](on-call-assistant-bot/README.md) · [PDF](on-call-assistant-bot/on-call-assistant-bot-case-study.pdf) |
| [Warehouse cost and change-control tool](#cost-and-change-control-tool-for-a-bigquery-data-warehouse) | An internal web tool that safely removes unused tables, applies the provider's clustering suggestions and puts schema changes behind an approval step. | [case study](warehouse-cost-and-change-control/README.md) · [PDF](warehouse-cost-and-change-control/warehouse-cost-and-change-control-case-study.pdf) · [appendix](warehouse-cost-and-change-control/technical-appendix.md) |
| [Verified LLM query rewrites](#verified-llm-rewrites-of-expensive-warehouse-queries) | A pipeline that has a language model rewrite expensive queries and accepts a rewrite only after a measured run proves it returns the same rows for less compute. | [case study](llm-query-optimisation/README.md) · [PDF](llm-query-optimisation/llm-query-optimisation-case-study.pdf) |
| [Feature-flag changes by AI agents](#feature-flag-changes-by-ai-agents-with-a-human-applying-every-change) | A review layer and agent tools that let AI agents read feature flags and propose changes, while only a person can apply one. | [case study](agent-feature-flag-change-requests/README.md) · [PDF](agent-feature-flag-change-requests/agent-feature-flag-change-requests-case-study.pdf) |
| [Feature-flag lifecycle](#a-lifecycle-for-feature-flags) | Expiry dates, stages computed from real usage, and a daily job that removes unused flags and tells their owners. | [case study](feature-flag-lifecycle/README.md) · [PDF](feature-flag-lifecycle/feature-flag-lifecycle-case-study.pdf) |

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

### Feature-flag changes by AI agents, with a human applying every change

**Read:** [case study](agent-feature-flag-change-requests/README.md) · [PDF, 2 pages](agent-feature-flag-change-requests/agent-feature-flag-change-requests-case-study.pdf)

**Role:** designer and sole developer · **Period:** March to November 2026 · **Status:** in production

**Stack:** Go, gRPC and Protocol Buffers, PostgreSQL, MCP (Model Context Protocol) tool server, Python web app

A review layer for a feature-flag service that holds more than 2,000 live flags. An AI coding agent can read any flag and propose a create, an update or a delete; the change is written only when a person applies it in the admin web app, after seeing the live and proposed configuration side by side. Proposals are stored apart from live flags, a stale proposal fails instead of overwriting a newer edit, and the agent tools have no way to approve anything.

What I personally built (all of it):

- The design of the review layer, end to end.
- In the flag service: one shared write path for hand edits and approved edits, the change request table and API, the apply transaction and the authorisation rules.
- The agent tools in a shared MCP tool server: read tools first, then propose and tracking tools.
- The review page in the admin web app, with the side-by-side diff, and the API contract for all of it.

The flag service's core and the tool server framework were built by colleagues.

### A lifecycle for feature flags

**Read:** [case study](feature-flag-lifecycle/README.md) · [PDF, 2 pages](feature-flag-lifecycle/feature-flag-lifecycle-case-study.pdf)

**Role:** designer and sole developer · **Period:** July 2025 to March 2026 · **Status:** in production since late 2025

**Stack:** Go, gRPC and Protocol Buffers, PostgreSQL, Slack, Python web app

A lifecycle for the same feature-flag service, where flags used to stay forever and every app fetched all of them. Each temporary flag now has an expiry date and a stage computed from evidence of real use: which app builds still contain it, and when the web last requested it. A daily job removes unused flags in slow, reversible steps and tells the owning team first. Apps fetch only the flags they use, which reduced the data sent to users' devices by about 10 TB.

What I personally built:

- The design of the lifecycle: the stages, the evidence rules and the safety rules.
- In the flag service: build tracking, usage tracking, the smaller fetch call with its fallback, the stage logic, expiry extensions, the daily job and the Slack notifications.
- The dashboards and lifecycle pages in the admin web app, and the API contract.

The flag service's core was built by colleagues. The mobile team owns the build pipeline that reports each build's flags and the code generator that reads the stage.

## Code walkthrough

The repositories are private, so the code is not here. I am happy to walk through the design and the code in a live session.
