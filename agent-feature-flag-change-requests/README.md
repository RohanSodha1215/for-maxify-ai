# Case study: letting AI agents change feature flags, with a human applying every change

*Also available as a [PDF](agent-feature-flag-change-requests-case-study.pdf).*

**Role:** designer and sole developer · **Period:** March to November 2026 · **Status:** in production

**Stack:** Go, gRPC and Protocol Buffers, PostgreSQL, MCP (Model Context Protocol) tool server, Python web app

## The problem

Our engineers use AI coding agents every day. Most features ship behind a feature flag: a server-side setting that decides which variant of a feature each user sees. The agent could write the code for a feature, but it could not see or change the flag that controls it. The engineer had to leave the session and copy the configuration by hand into an admin web page.

The obvious fix, giving agents write access, was not acceptable. The flag service holds more than 2,000 live flags and serves them to every app user. A flag change reaches production users directly, with no deployment step to catch a mistake.

## Background: the two systems this sits on

**The flag service.** An internal service, built by colleagues from 2019, that stores flags and decides each user's variant. It keeps all flags in memory to answer quickly. Engineers edit flags in an admin web app. I extended this service and its web app; I did not build the core.

**The tool server.** A shared server, built by our platform team, that exposes internal systems to AI agents as tools over MCP, a standard protocol for giving an agent tools. Each agent gets a fixed list of tools it may call. I added the flag tools to it.

## What I built

A review layer: an agent can read any flag and propose a change, and the change is written only when a person applies it.

```
AI agent, working for an engineer
      │  read tools · propose create / update / delete
      ▼
Tool server ── checks the proposal, records which engineer it is for
      │
      ▼
Flag service ── validates it, stores a change request
      │         (the live flags are not touched)
      ▼
Reviewer in the admin web app ── sees live and proposed side by side
      ├─ Apply    one transaction: claim the request, write the flag
      └─ Ignore   needs a comment, which the agent can read back
```

**How it works**

- Ten agent tools: five read flags, namespaces and targeting options; three propose a create, an update or a delete; two list and fetch change requests.
- A change request is a row in its own table: the target flag, the operation, the full proposed configuration, a required reason, who asked, and the version of the flag it was based on.
- The service validates a proposal when it arrives and again when it is applied, because other things can change in between.
- The review page shows the live configuration next to the proposed one, with changed lines highlighted. It fetches the live side when the reviewer opens the request, not when the request was filed.
- Each flag can have only one pending request, enforced by a unique index.

**Example.** An engineer tells their agent: "raise the new-checkout flag from 10% to 50% of users". The agent reads the flag, edits the percentage and calls the propose-update tool with a reason. The service stores it as a pending request, and the agent replies that the request is waiting for review and names the team that owns the flag. A reviewer opens it, sees one changed line, and presses Apply. If someone had edited the flag in the meantime, Apply fails with "the flag has changed" and nothing is written.

## Design decisions

- **Agents propose; only a person applies.** Agents sign in as service accounts, and the service refuses Apply and Ignore from a service account. It also refuses them from the person who filed the request. The tool server has no apply tool at all, and a test fails the build if one is ever added.
- **Proposals never enter the live flag table.** The service loads that whole table into memory to serve users, so a draft row there could reach production. Drafts live in a separate table.
- **A stale proposal fails; it never overwrites.** An update carries the version it was based on. Apply writes with a compare-and-swap on that version, so it cannot silently revert someone else's newer edit.
- **Apply is one transaction that claims the request first.** If two reviewers press Apply together, the second gets "already decided" before any flag is touched. If the flag write fails, the approval rolls back with it.
- **A hand edit and an approved edit run the same code.** I first moved validation and writes into one shared path, in a separate change that altered no behaviour, so the two routes cannot drift apart.
- **The server owns derived fields, and every request names a person.** The service computes a flag's type, stage and expiry date, and strips any values an agent echoes back. The engineer the agent works for is stored on the request, and each decision goes to the audit log.

## Results

- In production: agents can read every flag and propose changes to any of the 2,000+ live flags, and no agent can write one directly.
- The first agent-created flags exposed an old inconsistency in how platform names were stored (`iOS` and `ios`). I fixed it at the source: every write path now normalises and validates them.
- While building the propose tools I found that all agent calls shared one test credential, left from my first read-only version and exempt from write checks. I replaced it with a configured service account.

## What I personally built

- The design of the review layer, end to end.
- In the flag service: the shared write path, the change request table and API, the apply transaction, the authorisation rules, and the tests.
- The API contract (Protocol Buffers) for the flag service and for the agent tools.
- The agent tools in the tool server: the read tools first, then the propose and tracking tools.
- The review page in the admin web app, with the side-by-side diff.
- The fix for platform-name handling across all write paths.

The flag service's core and the tool server framework were built by colleagues.

## Why the code is not shared

The repositories belong to my employer and are private. I am happy to walk through the design and the code in a live session.
