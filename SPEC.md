# Blueprint

Blueprint is a CLI that turns an issue into a reviewed change. An agent writes
a plan, the owner approves it, an agent builds it, a second agent reviews it,
and the owner merges.

The same CLI runs from a terminal, from an agent session through a skill, or
from a CI job. Where issues and code live is up to plugins.

## Principles

- **The approved plan is the source of truth.** A change that has to deviate
  from it goes back to Planning for a new plan and a new approval.
- **Nothing runs until the owner says so.** There is no auto-start.
- **One status per issue.** That one value is all the state the flow needs.
- **The core knows nothing about hosts.** Issues live in memory. Plugins load
  and save them.
- **Less is the goal:** less code, fewer statuses, fewer moving parts.

## Layout

A monorepo:

| Package          | Holds                                                                      |
| ---------------- | -------------------------------------------------------------------------- |
| `core`           | The `blueprint` CLI, the flow, the in-memory issue store, stages, run records and hooks |
| `plugins/file`   | Issues as Markdown files in a repo                                         |
| `plugins/git`    | A local git code host: branches, merged by the owner                       |
| `plugins/github` | Issues, pull requests and a trigger on GitHub                              |
| `plugins/*`      | Other providers and extras, such as cost tracking or timeouts              |
| `skill`          | An agent skill that calls the CLI                                          |

## Concepts

| Term     | Meaning                                                                                                     |
| -------- | ----------------------------------------------------------------------------------------------------------- |
| Issue    | What the owner wants and how they'd know it's done, plus its status.                                        |
| Status   | Exactly one of: Backlog, Planning, Blocked, Building, Review, Done.                                          |
| Owner    | The person allowed to start stages and approve plans.                                                       |
| Stage    | One of three: plan, build, review.                                                                          |
| Run      | One execution of a stage on one issue. It has a log and an outcome, kept as a run record. Plugins can add to it, such as cost. |
| Result   | What every stage returns, checked against one schema.                                                       |
| Change   | What a build produces: a branch, plus a pull request if the code host has them.                             |
| Plugin   | A package that adds hooks, commands or a provider.                                                          |
| Provider | A plugin that stores issues or hosts code.                                                                  |

**Blocked** means waiting on the owner. The latest comment from Blueprint says
why: a question, a plan to approve, a failed review or a stop.

## The flow

```
Backlog → Planning → Blocked → Building → Review → Done
              ↑         │
              └─────────┘  answer and re-plan
```

1. **Plan.** The owner sets Planning. The plan stage reads the issue and the
   code, writes one plan with any questions, and sets Blocked. Every plan
   stops there.
2. **Answer or approve.** If the plan asks something, the owner edits the
   issue and sets Planning again. If the plan is ready, the owner sets
   Building. That is the approval.
3. **Build.** One model builds on a branch and opens a draft change.
4. **Review.** A different model reviews it. If it finds problems, the builder
   gets one repair round: it fixes the listed points on the same branch, and
   the reviewer checks again.
   - If the fix would break the plan, the issue goes back to Planning. The new
     plan stops at Blocked like any other, so it needs a new approval.
   - If review still fails, the change stays a draft and the issue goes to
     Blocked.
   - If review passes, the change is marked ready and the issue goes to Review.
5. **Merge.** The owner merges. A change merges itself only if it matches an
   allow list the owner sets and its checks pass. The code host's merge sets Done.

Rules that hold at every step:

- An issue has one run at a time.
- Every run can be stopped. A stop is recorded with its reason and sets
  Blocked.
- Only the owner's status changes start work. The one exception is a review
  sending an issue back to Planning, which can only end at Blocked.

## Core

- **Issues live in memory.** The core loads them from issue providers and
  writes status changes and comments back through them. With no provider, an
  issue lasts only as long as the process, which suits tests and one-off runs.
- **Prompts ship with the core**, one per stage. A repo can add prompt files
  of its own; they are files, not code.
- **Runs get an allowlisted environment.** No ambient key or token reaches an
  agent unless the run declares it.

## Plugins

The CLI is built on [oclif](https://oclif.io). Its plugin system is
Blueprint's plugin system: a plugin can listen to hooks and add commands.

**Hooks.** The core fires a hook before and after each run and each stage,
with the run record. A cost plugin adds up usage after each run. A timeout
plugin stops a run. Every run carries a stop signal, so a plugin can end it
at any time.

**Issue provider.** It must:

- list issues and read one
- add a comment
- read and set an issue's status

It may also say who last set the status. Without that, whoever runs the CLI
counts as the owner.

**Code host.** It must make a branch, report whether checks passed, and
merge. If it has pull requests, it also opens a draft and marks it ready.

**Trigger** (optional). It calls the CLI when the owner sets Planning or
Building, for example a CI job that runs on a GitHub label.

### File provider

Each issue is a Markdown file. Its frontmatter holds the status:

```yaml
---
status: Planning
---
```

Setting Planning or Building starts that stage when the CLI runs.

### GitHub provider

A GitHub issue has no status field, so the provider maps status onto two
labels and facts it can read:

| Status   | On GitHub                                                                         |
| -------- | --------------------------------------------------------------------------------- |
| Backlog  | Open, no Blueprint comment, no label                                              |
| Planning | `blueprint:plan` is on                                                            |
| Blocked  | Open, no label, no ready PR, latest Blueprint comment is a plan, failed review or stop |
| Building | `blueprint:build` is on                                                           |
| Review   | Open, non-draft PR from the issue's branch, no label                              |
| Done     | Closed                                                                            |

The two labels are never on an issue together. `blueprint:plan` comes off
when the plan ends; `blueprint:build` stays on through build and review and
comes off when the review outcome is set. Its trigger is a workflow that runs the CLI when the
owner adds a label. It needs no project board, no webhook and no organization.

## Dashboards

A dashboard is a consumer. It reads issues, statuses and run records through
the CLI, so it works with any provider. It can set the same statuses, close
issues and stop runs, and nothing else. It keeps no state it can't rebuild.

## What Blueprint does not do

- **No chat, coordinator or threads.** Agent chat tools already do that.
- **No stages beyond plan, build and review.** No research, memory, critique
  or retro stages.
- **No auto-start.** No loop picks up issues on its own.
- **No server in the core.** No webhook receiver and no hosted service. A
  trigger is a provider's job.
- **No deny lists.** Auto-merge only matches an allow list.
- **No review loops.** One repair round, then the owner decides.
- **No replay of stopped runs.** A stopped run says why; the owner restarts
  it.
- **No second approval path.** A button, a label and a status change all mean
  the same thing.
- **No provider state in the core.** Labels, boards and fields belong to their
  provider.
- **No built-in cost tracking, limits or notifications.** Those are plugins.

## Open questions

- **Locking.** One run per issue needs a lock that a laptop and a CI job both
  respect. It should come from the provider.
- **Run records.** Do they live in the repo, or only on the machine that ran
  them?
- **Owner without a host.** On a shared repo of files, anyone who can edit a
  file can approve a build. The owner may need to be named in config.
- **Per-repo plugins.** oclif plugins install per user. A repo may need a way
  to declare the plugins it expects.
- **Moving issues between providers.** An issue may start as a file and later
  move to GitHub. Only one copy should stay live.
- **Cost.** A terminal, an agent session and a CI job report usage
  differently. Cost needs one definition.
