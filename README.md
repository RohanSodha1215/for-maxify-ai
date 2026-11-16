# Representative work

Most of my professional work is in private repositories that belong to my employer, so I cannot share the code. This repository holds anonymized case studies of that work instead.

Each case study explains the problem, what I built, how it works, the main design decisions, and which parts were mine. Internal names and data are left out.

## Case studies

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

## Code walkthrough

The repositories are private, so the code is not here. I am happy to walk through the design and the code in a live session.
