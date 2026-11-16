# Case study: a lifecycle for feature flags, so that unused flags remove themselves

*Also available as a [PDF](feature-flag-lifecycle-case-study.pdf).*

**Role:** designer and sole developer · **Status:** in production since late 2025

**Stack:** Go, gRPC and Protocol Buffers, PostgreSQL, Slack, Python web app

## The problem

Our apps (iOS, Android, web and mobile web) ship features behind feature flags: server-side settings that decide which variant of a feature each user sees. A flag is meant to be temporary. In practice nobody removed them. A flag had no end date and no stage, and nobody could tell whether any app still read it.

This cost us in two ways. Every app fetched the full list of flags for its platform on each refresh, including flags its code no longer used, so users paid in data and waiting time. And engineers had no safe way to clean up: deleting a flag that an old app version still read could break that version.

## Background: the flag service

An internal service, built by colleagues from 2019, stores the flags and decides each user's variant. Today it holds more than 2,000 live flags, grouped into namespaces that teams own. Each flag carries platform tags that say which apps receive it. The mobile apps generate a typed code stub for each flag at build time; the build pipeline and that generator belong to our mobile team. I extended the service and its admin web app; I did not build the core.

## What I built

A lifecycle for every temporary flag: an expiry date, a stage computed from evidence of real use, and a daily job that cleans up and tells the owners.

```
Mobile build pipeline ── "build N contains flags A, B, C" ──┐
Serving path ── "flag A was just served on web" ────────────┤
                                                            ▼
                                             usage evidence per flag
                                                            │
                                               Daily maintenance job
        ├─ remove a mobile tag once the newest build no longer contains the flag
        ├─ remove a web tag after 24 hours without a request
        ├─ soft-delete flags that have no tags and no use left
        ├─ tell the owning team in Slack, 7 days before anything is deleted
        └─ save daily counts per namespace for the dashboards
```

**The stages**

| Stage | When | Mobile code stub |
|---|---|---|
| Active | before the expiry date | generated |
| Deprecated | after expiry, for a set number of days | generated, marked deprecated |
| Pending removal | still tagged for a platform, or still read on web | not generated |
| Ready for removal | no platform tags, no web use in 24 hours | not generated |
| Soft-deleted | after a further period with no use at all | hidden; the row is kept |

**How it works**

- **Mobile evidence.** Each app build reports the flags it was compiled with, and the service keeps the build range per flag and platform. A flag missing from the newest build has left the app's code.
- **Web evidence.** The serving path records, per platform, when each flag was last served.
- **A smaller fetch.** A new fetch call lets an app ask only for the flags its build contains. If the list is empty or names an unknown flag, the service falls back to the flags mapped to that build, then to all flags.
- **Extensions.** An owner can extend an expiry with a reason. Each extension is logged.
- **Dashboards.** Admin web pages show each team's flags by stage and type, with trends over time.

**Example.** A team finishes an experiment and removes it from the iOS and Android code. The next builds report their flag lists without it. Once the flag has expired and its deprecation period has passed, the daily job removes both mobile tags, and the web tags after 24 hours without a request. The owning team gets a Slack message with the deletion date. Seven days later, with still no use, the job soft-deletes the flag.

## Design decisions

- **The stage is computed, never stored.** It is derived on each read from the expiry date, the tags and the usage evidence. It cannot go stale, and nobody can set it by hand.
- **Remove only on evidence.** A mobile tag goes only when the newest build provably lacks the flag. A web tag goes only after measured silence. The job never guesses from a flag's age alone.
- **Every destructive step is slow and reversible.** Deletion is soft, comes after a waiting period, and is announced to the owning team first. Destructive steps are off in staging, and one switch disables the lifecycle.
- **The serving path never fails closed.** When the smaller fetch cannot be answered exactly, the service returns more flags, not fewer. A circuit breaker guards the fallback, and a metric counts each use.
- **Usage tracking stays off the request path.** Requests only set an in-memory mark. The database sees one batched write per minute, not one write per request.
- **Owners come from the team directory.** Notifications look up a namespace's owner in the company's team directory at send time, so they reach the current team.

## Results

- In production since the second half of 2025, across Android, iOS, web and mobile web.
- Every temporary flag now has an expiry date, a stage and an owning team. The daily job removes unused flags without manual work.
- Apps now fetch only the flags they use. Together with the cleanup, this reduced the data sent to users' devices by about 10 TB, and shortened the fetch that users wait on.

## What I personally built

- The design of the lifecycle: the stages, the evidence rules and the safety rules.
- In the flag service: build tracking, usage tracking, the smaller fetch call with its fallback, the stage logic, expiry extensions, the daily job, Slack notifications, daily snapshots, the tests, and the API contract (Protocol Buffers).
- The dashboards and lifecycle pages in the admin web app.

The flag service's core was built by colleagues. The mobile team owns the build pipeline that reports each build's flags and the code generator that reads the stage.

## Why the code is not shared

The repositories belong to my employer and are private. I am happy to walk through the design and the code in a live session.
