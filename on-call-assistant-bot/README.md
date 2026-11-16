# Case study: an on-call assistant bot for a data engineering team

*Also available as a [PDF](on-call-assistant-bot-case-study.pdf).*

**Role:** designer and main developer · **Status:** in production since October 2026

**Stack:** Python, an LLM agent framework (steps defined in YAML), Slack, MCP tool servers, sandboxed scripts

## The problem

Our data engineering team runs an on-call rota. Colleagues from other teams ask the on-call engineer for help in Slack: "how do I get access to this table?", "please add a column", "why did my job fail?".

I analysed 12 months of these requests (568 in total, about 50 a month). Most were repeats: questions already answered many times, or standard requests where the engineer only copied details into a form. Each one still interrupted a person.

## Background: the two platforms the bot is built on

Both are internal platforms built by colleagues. I did not build them; I built the bot on top of them.

**An agent framework.** It lets a team build a chat agent without writing application code. An agent is a folder of YAML files that describes a graph of steps:

- a **model step**: a prompt for a language model, with optional tools and a fixed output format;
- a **router**: a fixed condition that picks the next step;
- a **script step**: a short script run in a sandbox;
- a **human step**: pauses the run until a person replies or presses a button;
- a **fixed-text step**: posts a message with no model involved.

The framework runs the graph, connects it to Slack, and saves the state so a run can wait for a person and continue later.

**A knowledge registry.** It stores curated content for AI agents as a tree of entries. Each entry has a name, a short description and a body. An agent first reads the index (names and descriptions only), then fetches just the entry it needs. A change to an entry is proposed and reviewed before it goes live.

## What I built

A Slack bot that reads each request and does one of three things:

```
Request in Slack
      │
   Triage ── reads the registry index, picks at most one entry
      │
      ├─ Answer      reply from that one knowledge entry
      ├─ Prepare     fill in a request template, show a preview,
      │              send it only after the person presses Confirm
      └─ Escalate    tag the on-call engineer with a fixed message
```

**How it uses the two platforms**

- The bot is one agent on the framework: about 35 step files and no application code.
- Its knowledge lives in the registry in two folders: FAQ entries and request templates. Each description lists the wordings it covers and the requests it excludes, and triage routes on that.
- To answer, the bot fetches one entry and replies only from its text.
- To prepare a request, a model step fills in the template, a script step checks the draft against the template's rules, and a human step waits for Confirm.
- Updating an answer or adding a request type is an edit in the registry. The bot is not redeployed.

**Example.** A colleague writes "I need read access to the storage bucket X for my account". The bot drafts the access request, shows it with Confirm and Cancel buttons, and files it only when the colleague confirms. The on-call engineer just approves it.

The bot also investigates production alerts and failed data jobs, and posts what it finds for the on-call engineer.

## Design decisions

- **A person confirms before anything is sent.** The bot prepares requests; it never approves or executes them.
- **Fixed rules decide the risky steps.** A language model reads the request and writes the draft. Plain code validates the draft, sends it, and decides when to page a person.
- **Answers come only from curated knowledge.** If no entry covers the question, the bot escalates. It does not answer from the model's general knowledge.
- **New request types need no deployment.** A request type is a template in the registry, not code.
- **A wrong escalation is harmless; a wrong action is not.** When unsure, the bot hands over to a person.

## Results

- About a third of the 568 analysed requests fall into categories the bot now answers or prepares.
- Three request types are live with the confirm step: access requests, adding columns, and creating tables.
- I measured the bot after launch. Most requests it still escalated were code-review requests (29% of recent volume) and genuine judgement calls. That finding set the next piece of work: giving the bot read access to pull requests so it can pre-check them.

## What I personally built

- The bot's overall design, from its first version to the production rollout.
- The triage, answer, request-preparation and escalation flows.
- The design of the alert-investigation and job-failure flows, which a teammate implemented.
- An API on our internal access-request service so that automated agents can file and read requests but never approve them.
- The knowledge base: the FAQ entries and request templates.
- The request analysis that decided what to build.
- A fix in the shared framework for Slack buttons that failed to render.

## Why the code is not shared

The repository belongs to my employer and is private. I am happy to walk through the design and the code in a live session.
