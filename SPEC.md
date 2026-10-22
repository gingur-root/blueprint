# Blueprint

Blueprint is a CLI that turns an issue into a reviewed change. An agent writes
a plan, the owner approves it, an agent builds it, a second agent reviews it,
and the owner merges.

The same CLI runs from a terminal, from an agent session through a skill, or
from a CI job.

## Why

Blueprint replaces devkit's agent pipeline: `bin/agent`, the dashboard, the
webhook receiver and the `gingur-root/pipeline` caller. Its first user is a
product repo, not the factory's own repos. The old pipeline is deleted once
Blueprint has taken one issue there from plan to merge. Until then, nothing new
goes into the old pipeline.

## Principles

- **The approved plan is the source of truth.** A change that has to deviate
  from it goes back to Planning for a new plan and a new approval.
- **Nothing runs until the owner says so.** There is no auto-start.
- **One status per issue.** That one value is all the state the flow needs.
- **The core knows nothing about hosts.** Issues live in memory. A provider
  loads and saves them.
- **Less is the goal:** less code, fewer statuses, fewer moving parts.

## Layout

| Package  | Holds                                                                         |
| -------- | ----------------------------------------------------------------------------- |
| `core`   | The `blueprint` CLI, the flow, the in-memory issue store, stages and run records |
| `github` | The GitHub provider: issues, pull requests and the label trigger              |
| `skill`  | An agent skill that calls the CLI                                             |

## Concepts

| Term     | Meaning                                                                                  |
| -------- | ---------------------------------------------------------------------------------------- |
| Issue    | What the owner wants and how they'd know it's done, plus its status.                     |
| Status   | Exactly one of: Waiting, Planning, Building, Done.                                       |
| Owner    | The person allowed to start stages and approve plans.                                    |
| Stage    | One of three: plan, build, review.                                                       |
| Run      | One execution of a stage on one issue. It has a log and an outcome, kept as a run record. |
| Result   | What every stage returns, checked against one schema.                                    |
| Change   | What a build produces: a branch and a draft pull request.                                |
| Provider | A module that stores issues and hosts code.                                              |

**Waiting** means waiting on the owner. The latest comment from Blueprint says
why: a question, a plan to approve, a ready change to merge, a failed review or
a stop. A new issue with no Blueprint comment is Waiting too.

## The flow

```
Waiting → Planning → Waiting → Building → Waiting → Done
              ↑          │
              └──────────┘  answer and re-plan
```

1. **Plan.** The owner sets Planning. The plan stage reads the issue and the
   code, writes one plan with any questions, and sets Waiting. Every plan stops
   there.
2. **Answer or approve.** If the plan asks something, the owner edits the
   issue and sets Planning again. If the plan is ready, the owner sets
   Building. That is the approval.
3. **Build.** An agent builds on a branch and opens a draft change.
4. **Review.** A separate run with a fresh context reviews the change. A
   change passes only when the review passes and its checks are green; a red
   check counts as a failed review. On a failure the builder gets one repair
   round on the same branch, and the review runs again.
   - If the fix would break the plan, the issue goes back to Planning. The new
     plan stops at Waiting like any other, so it needs a new approval.
   - If it still fails, the change stays a draft and the issue goes to
     Waiting.
   - If it passes, the change is marked ready and the issue goes to Waiting.
5. **Merge.** The owner merges. A change merges itself only if it matches an
   allow list the owner sets and its checks pass. The merge sets Done.

Rules that hold at every step:

- An issue has one run at a time.
- Every run has a time limit and runs with no terminal input, so an agent that
  waits for input fails instead of hanging. The core sets a default limit; a
  repo can change it.
- Every run can be stopped. A stop is recorded with its reason and sets
  Waiting.
- Only the owner's status changes start work. The one exception is a review
  sending an issue back to Planning, which can only end at Waiting.

## Core

- **Issues live in memory.** The core loads them from the provider and writes
  status changes and comments back through it. With no provider, an issue
  lasts only as long as the process, which suits tests and one-off runs.
- **Prompts ship with the core**, one per stage. A repo can add prompt files
  of its own; they are files, not code.
- **Runs get an allowlisted environment.** No ambient key or token reaches an
  agent unless the run declares it.
- **Tests drive whole runs.** They run each stage end to end against the
  in-memory store and a stub agent, rather than testing internals.

## Provider

A provider must:

- list issues and read one
- add a comment
- read and set an issue's status, and say who last set it
- make a branch, open a draft pull request, mark it ready, report whether its
  checks passed, and merge

### GitHub

A GitHub issue has no status field, so the provider maps status onto two
labels:

| Status   | On GitHub                 |
| -------- | ------------------------- |
| Waiting  | Open, no Blueprint label  |
| Planning | `blueprint:plan` is on    |
| Building | `blueprint:build` is on   |
| Done     | Closed                    |

The two labels are never on an issue together. `blueprint:plan` comes off when
the plan ends; `blueprint:build` stays on through build and review and comes
off when the review outcome is set.

The trigger is a workflow that runs the CLI when the owner adds a label. Its
concurrency group is the issue number, which gives one run per issue. Each
run's record is the workflow run, linked from Blueprint's comment on the issue.
It needs no project board, no webhook and no organization.

The issue list filtered by label is the dashboard.

## What Blueprint does not do

- **No chat, coordinator or threads.** Agent chat tools already do that.
- **No stages beyond plan, build and review.** No research, memory, critique
  or retro stages.
- **No auto-start.** No loop picks up issues on its own.
- **No server.** No webhook receiver, no hosted service and no dashboard app.
- **No deny lists.** Auto-merge only matches an allow list.
- **No review loops.** One repair round, then the owner decides.
- **No replay of stopped runs.** A stopped run says why; the owner restarts
  it.
- **No second approval path.** A label and a status change mean the same
  thing.
- **No plugin system yet.** Cost tracking, notifications and other providers
  wait until a real repo needs them. The provider interface is the one seam.

## Open questions

- **Owner on a shared repo.** GitHub says who added a label. Is the repo's
  admin the owner, or is the owner named in config?
</content>
</invoke>
<invoke name="Bash">
<parameter name="command">cd /tmp/claude-1000/-home-trhinehart-github-gingur/28fd054e-3e6c-555f-9389-007d1559dd5c/scratchpad/blueprint && git diff --stat && git add SPEC.md && git commit -q -m "docs(spec): trim Blueprint to GitHub, four statuses and a checks gate

Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>" && git rev-parse --abbrev-ref HEAD && git push -q -u origin docs/tenth-man-trims 2>&1 | grep -v remote; cat > ../bp-pr.md <<'EOF'
<!-- claude-projects-attribution: {"github_login":"gingur"} -->
_Requested by **Troy**_

Before: the spec plans three providers, an oclif plugin system, six statuses inferred from GitHub comments and PR state, a dashboard app, and six open questions.

After: GitHub is the only provider, there are four statuses mapped onto two labels, a red check fails review, every run has a time limit with no terminal input, and the spec says Blueprint replaces devkit's pipeline and starts on a product repo.

How: a tenth man pass (YAGNI, serve the 80%) against the [platform verdict](https://claude.ai/artifact/MyW5zTEHXGbnc9c6qtDrpy). Each change is listed in the project thread so any can be dropped. Prose only; checked by reading the diff.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
gh pr create --draft -R gingur-root/blueprint --head docs/tenth-man-trims --title "docs(spec): trim Blueprint to GitHub, four statuses and a checks gate" --body-file ../bp-pr.md