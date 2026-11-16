# Case study: verified LLM rewrites of expensive warehouse queries

*Also available as a [PDF](llm-query-optimisation-case-study.pdf).*

**Role:** designer and sole developer · **Period:** May 2026 to present · **Status:** in production

**Stack:** Python (Flask), PostgreSQL, Vue 3, BigQuery Python SDK, LLM APIs via an OpenAI-compatible proxy

## The problem

Our company's data warehouse bills for compute: every query costs according to how much processing it needs. Many of the most expensive queries are scheduled jobs that rebuild tables every day, written by many teams over several years. Some read far more data than they need, or do their work in an expensive order. Because they run daily, each inefficiency is paid again every day.

Fixing one is skilled and slow. An engineer has to understand the query, rewrite it, and then prove that the new version returns the same result and really is cheaper. A language model can propose a rewrite in seconds, but its output cannot be trusted: it may not be valid SQL, it may return different data, or it may cost more. The problem was to turn fast, untrusted suggestions into changes that an owning team would accept.

## Background: four warehouse features the tool relies on

The warehouse is Google BigQuery. Four of its standard features matter here:

- **Compute is measured in slot time.** A slot is a unit of processing capacity, and a query's slot time is how much of it the query used. The warehouse also reports the bytes a query read. Our bill follows compute, so slot time is the number the tool tries to reduce.
- **A dry run** validates a query and reports which tables it reads and how many bytes it would read, without running it and without cost.
- **Time travel** lets a query read a table as it was at a moment in the last few days, by adding `FOR SYSTEM_TIME AS OF <timestamp>` after the table name.
- **Job statistics.** Every finished query has recorded statistics: slot time, bytes read, bytes shuffled between stages, and a per-stage execution plan.

## What I built

A pipeline with a web interface that has a language model propose a cheaper version of a query, and accepts the proposal only after the warehouse itself has shown that it is valid, returns the same number of rows and uses less compute.

```
Pasted SQL, or a past job with its recorded statistics
      │
Collect context ──── a dry run lists the tables; fetch their definitions and sizes
      │
Model proposes a rewrite ◀──┐
      │                     │ the warehouse's error message
Dry run of the rewrite ─────┘ (syntax errors only, limited retries)
      │ valid
Run on one data snapshot ─── both queries, or the rewrite only against the
      │                      recorded baseline; cancelled once it costs more
Verdict ──────────────────── row counts match AND compute went down
      │ passed
Hand-over ────────────────── a person sends it to the owning team
```

**How a run works**

A user starts a run in one of two ways: they paste a query, or they press "Optimise query" on a finished job in the tool's job history. The run then goes through four stages as a background job, and the page shows the current stage live.

1. **Collect context.** The tool dry-runs the query and takes the list of tables from the warehouse's own parser. It does not parse SQL itself: a regular expression cannot reliably tell a table name from a comment, a string or a named subquery. For each table it fetches the table definition, row count and size.
2. **Ask the model.** The prompt holds the query, the table context, optional notes from the user, and a rule file that describes rewrite techniques and the constraints a rewrite must keep. The model must reply in a fixed JSON format: the new query, a list of changes, and its reasoning.
3. **Dry-run the rewrite.** If the warehouse rejects it, the tool sends the model a correction prompt with the failed query and the exact error message, up to a configurable number of retries. Only syntax errors are retried. The original query is dry-run too; if the original itself is invalid, the run stops there.
4. **Run and compare.** The tool runs the queries on the same snapshot of the data, measures them, and decides. This stage is described next.

**Running the comparison safely**

This stage executes real queries, so it is built to give a fair result without becoming a cost of its own.

- **Same data for both queries.** Every table reference in both queries is rewritten to read the tables as of one timestamp, taken 10 seconds in the past to absorb clock drift. Data that lands between the two runs therefore cannot change the result. The rewrite has to put the time-travel clause after the table's alias, which is where the warehouse requires it, and it handles the five ways a table can be written in SQL. The rewritten queries are dry-run first; if they fail, the tool compares without pinning and records that it did.
- **Real cost, not cached cost.** The result cache is switched off, so a query cannot look free because it ran recently. Each query writes to a temporary table, which avoids the size limit on returned results and gives an exact row count from the table's metadata. The table is deleted afterwards.
- **A rewrite never costs more than the original.** While the rewrite runs, the tool polls its slot time every few seconds and cancels it as soon as it reaches the original's total. A rewrite that is going to lose is stopped at the break-even point.
- **Half the cost when starting from a past job.** The original already ran in production, so its recorded statistics become the baseline and only the rewrite is executed. It is pinned to the moment the original job started, so both read the same data. The tool always takes the query text from the job record, never from the form, so the statistics and the query cannot belong to different versions.
- **One verdict rule, chosen in configuration.** Row counts must match. Then slot time must fall, or bytes read must fall, or both, depending on a setting. Bytes shuffled are reported for information but do not decide the verdict.
- **Bounded in time.** Each query has a timeout, and a user can abort a run at any point. Both cancel the running warehouse job and remove its temporary table.

**The model client**

- **Two providers behind one interface:** an OpenAI-compatible proxy that reaches several vendors' models, and a direct cloud API. Provider, model, address, timeout, retries and output limit are all settings in the UI. A change takes effect on the next call without a restart: each client rebuilds itself when the settings it was built from change.
- **Structured output.** The JSON format is sent as a strict schema. For models that do not support strict schemas, a setting switches to plain JSON mode, and the parser also accepts JSON wrapped in a code block.
- **Failures that say what went wrong.** A reply cut off by an output limit is reported as exactly that. One proxy returned such a reply as an empty `{}` with a normal finish reason, which looked like a model that ignored the format. Newer models that reject the temperature setting are retried once without it. A rejected key is reported as a configuration problem, not as a model failure.
- **Cost attribution.** Every model call carries labels for the use case and the step (first attempt or correction retry), so model spend can be traced back to this tool.

**Prompts as configuration.** The prompt and the rule file are files selected on the configuration page. Three strategies exist today. Trying a new one is a new file and a setting change, not a release. The configured paths must stay inside the tool's own folder, so a setting cannot point the tool at an arbitrary file on the server.

**Operating it.** Runs are background jobs. Up to three run in parallel, which is configurable, while the tool's other background work stays one job at a time. If a run cannot be scheduled, its placeholder is marked failed at once instead of showing "Running" forever. A command-line version runs the same pipeline without the web server or database, for trying prompts and models locally.

**Hand-over.** The tool never changes a production job itself. A user selects verified results, groups them under headings in the order they choose, and sends one chat message to the owning teams. Only results that passed the comparison with lower slot time can be sent. Each line names the pipeline and task that runs the query, taken from the original job's labels, shows the slot hours before and after, and links to the full evidence; the message ends with the total saved. The tool records who sent it, when, and to which teams.

**Example.** A scheduled task rebuilds a daily report and reads an entire events table, filtering to yesterday's rows only at the end. A user opens the job in the job history and presses "Optimise query". The tool loads the recorded statistics and fetches the definitions of the tables the job reads. The model moves the date filter next to the table read and selects only the columns the report uses. Its first answer refers to a column alias that does not exist; the dry run fails, the error goes back to the model, and the second answer passes. The rewrite runs pinned to the moment the original job started, returns the same number of rows, and uses less slot time, so it passes. The user sends it to the owning team, and the message names the pipeline task to change.

## Design decisions

- **The model proposes; the warehouse decides.** A rewrite is accepted only on the warehouse's own evidence: a dry run, then a measured run. No language model judges another model's output.
- **Use the warehouse's parser for anything structural.** The list of tables comes from a dry run. The one place the tool edits SQL text, the time-travel rewrite, is checked by a dry run before use.
- **Compare like with like.** Both queries read the same snapshot with the cache off, so the difference measured is the query's, not the data's or the cache's.
- **Verification must never cost more than it can save.** Early cancellation, reuse of recorded baselines and per-query timeouts cap what a failed attempt can spend.
- **Retry only what the model can fix.** A syntax error is fed back with the exact message. A rewrite that is valid but slower is a wrong idea, not a typo, so it is not retried automatically.
- **A person ships the change.** The tool produces evidence and a message; the owning team decides and applies it.

## Known limits and next step

- The verdict compares row counts, not row contents, so a person still reads the rewrite before adopting it.
- Time travel only reaches back a few days. For older jobs, and whenever the pinned queries fail their dry run, the comparison runs without pinning, and the result records this.
- **In progress:** giving the model the original query's per-stage execution plan, so it targets the stage where slot time is actually spent instead of inferring it from table definitions. For pasted queries this needs one run of the original up front, and that run's statistics are reused as the baseline, so the original is still executed only once.

## Results

- The tool does not change production jobs itself; owning teams apply the rewrites it verifies. Those rewrites have saved about USD 4,000 a month.
- Every rewrite that reaches a team has passed a measured run against the original, and the message carries the before-and-after numbers.

## What I personally built

All of it, from the first version to production:

- The pipeline: context collection, prompt assembly, the correction loop, the time-travel rewrite, the measured comparison with early cancellation, timeouts and abort.
- The past-job mode with its recorded baseline, and the "Optimise query" button that starts it.
- The model client with two providers, structured output, settings that apply without a restart, and failure detection.
- Parallel runs in the background job executor, and the command-line tool.
- The hand-over messages to owning teams, and the web pages for starting, following and reviewing runs.

The job history that past-job runs start from is a separate feature of the same tool.

## Why the code is not shared

The repository belongs to my employer and is private. I am happy to walk through the design and the code in a live session.
