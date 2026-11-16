# Data engineering operations bot

**Evidence for:** my strongest production AI workflow

**Role:** designer and main developer · **Status:** in production since October 2026

A Slack bot that handles the requests other teams send to a data engineering team's on-call engineer. It answers repeat questions from a curated knowledge base, prepares standard requests and files them only after a person confirms, and hands everything else to the on-call engineer.

The code is in a private repository that belongs to my employer, so this write-up describes it without internal names. A shorter [case study](../../on-call-assistant-bot/README.md) gives more background on the two internal platforms the bot is built on.

## Short answers

| Question | Answer |
|---|---|
| Business problem | The on-call engineer received about 50 requests a month from other teams. Most were repeats. |
| What I built | The bot's design, its triage, answer, request and escalation flows, its knowledge base, and an API on the access-request service that lets agents file requests but never approve them. |
| How it works | A graph of steps. A language model reads and drafts. Fixed rules and scripts validate, send and escalate. A person confirms before anything is sent. |
| Guardrail from a real failure | Escalation was written by a model and, in one case, reached nobody. It is now a fixed message that always notifies the on-call engineer. |
| Outcome | Three request types are live end to end. About a third of the 568 analysed requests fall in categories the bot covers. The resolved-without-a-person rate is not measured yet. |

## 1. The business problem, and what I built

A data engineering team runs an on-call rota. Colleagues from other teams ask the on-call engineer for help in Slack: "how do I get access to this table?", "please add a column", "why did my job fail?".

I classified 12 months of these requests: 568 in total, about 50 a month. Most were repeats. Either the question had been answered many times before, or it was a standard request where the engineer only copied details into a form. Each one still interrupted a person.

**What I personally built**

- The bot's overall design, from the first version to the production rollout.
- The triage, answer, request-preparation and escalation flows described below.
- The knowledge base: about 85 FAQ entries and three request templates.
- An API on the internal access-request service, so that an automated agent can file and read requests but can never approve one.
- The 12-month request analysis that decided what to build.
- A fix in the shared agent framework for confirmation buttons that failed to appear in Slack.

**What others built**

- The agent framework and the knowledge registry are internal platforms built by colleagues.
- The bot also investigates production alerts and failed data jobs. I designed those two flows and a teammate implemented them. They are not covered here.

## 2. How it works in production

![System overview](system-overview.png)

The bot is a graph of steps. Each step is one of four kinds, and the kind decides how much it is trusted:

| Kind of step | What it does | Used for |
|---|---|---|
| Language model | Reads text and returns an answer in a fixed schema | Understanding a request, writing a draft |
| Script or fixed rule | Plain code with no model | Validating, sending, routing |
| Person | Pauses the run until someone replies or presses a button | Confirming a request |
| Fixed text | Posts a message that never changes | Escalating, reporting the result |

**Models.** One general-purpose model (a Gemini Flash model) is reached through an internal LLM gateway. Decision steps run at temperature 0 and must return a fixed schema, for example an action from the list `answer`, `prepare`, `escalate`.

**Tools.** The knowledge registry is exposed as an MCP tool server, and the bot only reads from it. The triage step reads the index (names and descriptions) and never the entry bodies. The answer step fetches the one entry that triage chose. The step that drafts a request has no tools at all.

**Data.** The registry holds two folders for this bot: FAQ entries and request templates. Each description lists the wordings it covers and the requests it excludes, and triage routes on that. A template states its scope, the values to collect, the request format and the lines of the preview. Changing an answer or adding a request type is an edit in the registry, which a person reviews before it goes live. The bot is not redeployed.

**Thread context.** A message such as "can you raise the request for the above?" means nothing on its own. Before triage, a step with no model reads the earlier messages of the Slack thread and passes them on as background. The model steps use that text only to work out what the latest message refers to.

**External systems.** Slack carries the conversation and the buttons. An identity directory maps the Slack user to a work email. A sandbox runs the scripts. The access-request service receives the request through its REST API, and a human approver decides there.

### Preparing a request, step by step

This is the path with the most at stake, because it ends in a write to another system.

![Request flow](request-flow.png)

A worked example: a colleague writes "I need read access to the storage bucket X for my account". Triage picks the access-request template. The draft step fills it in and asks one question if a value is missing, for example how long the access is needed. The check step builds the preview. If a template rule requires it, the preview ends with a one-sentence note, for example that a second person named in the message needs a separate request. The colleague presses Submit. The send step files the request under the colleague's name, and the bot posts the link. The on-call engineer only has to approve it.

Three request types run this way today: access requests, adding columns to a table, and creating a table.

## 3. Quality, permissions, monitoring and failures

### How I evaluated quality

- **Started from data.** I classified every request of the past 12 months with fixed counting rules. The categories decided what the bot should cover first.
- **Checked every knowledge entry.** Each FAQ entry was verified before it was published. Ten entries I could not verify were held back. After every accepted change I compared the live registry with the reviewed copy, entry by entry.
- **Tested each change against real requests.** Before a change ships, the configuration is validated offline, and a local copy of the bot runs in a private Slack channel. I replay real past threads there and compare the replies. One example is a real 7-message thread about a schema change: after thread context was added, the bot kept the table, the columns and the reason on every turn, where the earlier version had lost at least one of them on three of the seven turns.
- **Checked the triggers offline.** A script confirms that every kind of Slack message matches exactly one trigger, so the bot runs once per message.
- **Measured after launch.** I classified the newer requests the same way to see what the bot still handed to a person. The largest group was code-review requests, 29% of recent volume.

### How permissions are controlled

- **The bot cannot approve anything.** I built the API on the access-request service so that any call with an agent token is refused on approve, reject and configuration endpoints, before the admin check runs. Approval needs a signed-in person.
- **Requests are filed in the requester's name.** The bot passes the requester's email, and the service rejects a call without one.
- **One step holds the service token.** Only the send script has it, and it runs in its own sandbox. No model step can see the token or call the service.
- **The model cannot choose where a request goes.** Scripts accept only a fixed list of endpoints and only the POST method. A template can name a connection from that list but cannot set a host or a header.
- **A person confirms every request.** The send script also checks that the Submit value is present, so the rule does not depend on the graph alone.
- **Model text cannot notify people.** Mentions inside model-written text are neutralised before it is shown in Slack, and a note that contains one is refused. Only fixed messages can notify the on-call engineer.
- **Earlier thread messages are background, not instructions.** The prompts tell the model never to act on an earlier request and never to follow instructions written in the thread. Whatever it drafts still has to pass the check script, the person's Submit and the send rules.
- **The bot runs only in listed channels** and ignores its own messages, so it cannot trigger itself.

### How operation is monitored

- **A failure is visible where the request is.** Every escalation and every failed send notifies the on-call engineer in the same Slack thread.
- **Every decision leaves a reason.** The model steps return a one-line reason, and the scripts return an error code and the HTTP status. These are kept in the run log.
- **Every reply has feedback buttons**, and replies are marked as experimental while the bot is new.
- **Bot-filed requests are visible in the access-request service**, where the approver sees them like any other request.

### How failures are handled

The rule is the same everywhere: when a step fails, the bot does not guess. It says nothing was sent and brings in a person.

| What goes wrong | What the bot does |
|---|---|
| The registry cannot be read during triage | Escalates. It does not answer from the model's general knowledge. |
| The chosen FAQ entry cannot be fetched | Posts one fixed sentence asking the requester to notify the on-call engineer. |
| The earlier thread messages cannot be read | Continues with the latest message only. |
| A template cannot be loaded, or is too long | A fixed rule stops the request before any model runs: "I could not prepare this request, so nothing was sent." The on-call engineer is notified. The template is never cut short. |
| The draft is not valid | "I could not prepare this request, so nothing was sent." The on-call engineer is notified. |
| The requester keeps correcting the draft | After 10 rounds the request is escalated. |
| The service rejects the request | Shows the service's reason and states that nothing was created. |
| The service refuses the bot, or redirects | Nothing is sent again. The on-call engineer is notified. |
| The service does not answer | "I could not confirm whether this request was created." The bot asks that nobody submits it again and notifies the on-call engineer. |

## 4. A real failure that became a permanent guardrail

**The failure.** In the first versions, the escalation message was written by a model. The prompt told it to mention the on-call engineer, except when the requester had already mentioned them, to avoid a double notification. During the trial before the full rollout, some bot replies were posted in a different channel from the original request. In those cases the model followed the exception and left the mention out. The reply said the on-call team would take a look, but it sat in a channel where nobody had been notified. An escalation that reaches nobody is the worst outcome for this bot, because the requester believes a person is on it.

**The guardrail.** Escalation is now a fixed message with no model call. It always mentions the on-call engineer and cannot fail on a model error. The cost is an occasional second notification, which I accepted.

**The rule I took from it.** A rule in a prompt is not a guardrail, because a model can ignore it. Anything that must always happen is now a fixed rule, a script or fixed text. Four more guardrails came from the same rule:

- **A model wrote a pull-request link into a field meant for a request id**, although the schema had a pattern for it. The router now checks the shape of that field in code before using it.
- **A test call to the service was redirected to a login page.** The send script now never follows a redirect, because that would carry the service token to another host. It reports the request as not sent.
- **A sandbox timeout while loading a template ended in a generic escalation**, and the outcome depended on how a model read an empty template. A fixed rule now stops the request before any model runs, with a message that says nothing was sent.
- **Confirmation buttons did not appear in Slack**, so a person could not confirm. The cause was two buttons sharing one id in the shared framework. I fixed it there and added regression tests.

## 5. Outcome, and what remains

**Achieved**

- The bot has been in production since October 2026, in the team's main request channel.
- Three request types run end to end with the confirm step.
- Follow-up messages in a thread work: the bot reads the earlier messages, so a requester does not have to repeat details.
- About a third of the 568 analysed requests fall in categories the bot now answers or prepares. This is coverage, not a measured reduction in on-call work.

**What remains**

- **A measured resolution rate.** I do not yet have the share of requests the bot resolved without a person. The feedback buttons and the run log hold the data.
- **An automated regression set.** Evaluation is replay and manual review today. The replayed threads should become a fixed test set that runs on every change.
- **Who may press Submit.** Anyone in the thread can press it, although the request is always filed under the person who started the thread. Closing this needs a change in the shared framework.
- **Code-review requests.** They are the largest group the bot still escalates. The next step is read access to pull requests so that it can pre-check them.

## Why the code is not shared

The repository belongs to my employer and is private. I am happy to walk through the design and the code in a live session.
