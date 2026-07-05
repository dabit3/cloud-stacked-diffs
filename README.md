# cloud-stacked-diffs

Agent skill that turns big tasks into a reviewable stack of draft PRs. Works with Devin, Claude, Codex, and any harness supporting the [Agent Skills standard](https://agentskills.io/specification).

The skill is named `cloud-stacked-diffs` and lives at [`.agents/skills/cloud-stacked-diffs/SKILL.md`](.agents/skills/cloud-stacked-diffs/SKILL.md).

## Why

The most common failure mode of a cloud coding agent working async is delivering one giant, unreviewable PR. This skill teaches the agent to deliver multi-step work as a chain of stacked draft pull requests instead: one branch per step, each branching from and targeting the previous step's branch, so reviewers can approve the work in small increments and merge bottom-up.

Stacking also means waiting on human review never blocks the agent: each verified step is a solid foundation for the next, so the agent can work through an entire task as a chain of small, green, review-ready PRs while reviews happen on their own schedule.

## Install

With the [Skills CLI](https://skills.sh/):

```bash
npx skills add dabit3/cloud-stacked-diffs
```

Or copy the skill directory into your project manually:

```bash
mkdir -p .agents/skills
cp -r path/to/cloud-stacked-diffs/.agents/skills/cloud-stacked-diffs .agents/skills/
```

If you use Devin, connecting a repository that contains this skill is enough: skills in `.agents/skills/` are discovered automatically.

## Usage

Give your agent a multi-step task and ask for stacked PRs:

```
Split this into stacked PRs:
1. Scaffold the project
2. Add design tokens
3. Build the app shell
```

The agent produces a branch chain with one draft PR per step:

```
main
 └─ stack/webapp/01-scaffold           PR #1 (base: main)
     └─ stack/webapp/02-design-tokens  PR #2 (base: stack/webapp/01-scaffold)
         └─ stack/webapp/03-app-shell  PR #3 (base: stack/webapp/02-design-tokens)
```

It finishes with a summary of the branch chain and all PR links, in merge order.

### With a Linear MCP integration (optional)

If the agent also has an issue tracker connected, for example the [Linear MCP server](https://linear.app/docs/mcp), the tracker can drive the stack instead of an inline list:

```
Read ENG-123 in Linear and treat its sub-issues, in order, as the steps.
Split the work into stacked PRs, one per sub-issue. As you go, move each
sub-issue to In Progress when you start it and to In Review with the draft
PR link when its PR is up. When done, comment the full stack order and all
PR links on ENG-123.
```

The skill defines how the stack is built; the MCP adds read/write access to the board. Steps come from sub-issues, per-step status lives on the issues, and PR links land in Linear, so reviewers can pick up the stack from either GitHub or Linear. This is especially useful for cloud agents working async, where the tracker is the shared source of truth between the agent and the humans reviewing it.

## What the skill enforces

- One branch and one draft PR per step, named `stack/<stack-slug>/NN-step-slug` — the per-stack namespace keeps parallel stacks in one repository from colliding, and resuming a run verifies a pre-existing branch actually belongs to the stack before building on it
- Steps are split so each is one coherent concern that passes checks on its own; a step that can't go green in isolation is a bad split to reorder or combine
- Step 1 branches from the default branch (detected, not assumed to be `main`); every later step branches from the previous step's branch
- Repository checks (lint, typecheck, tests) pass before each PR is opened, and are re-run on every branch a restack touches, because verification is head-commit-specific
- PR wiring is verified after creation: the base actually points at the parent branch and the diff is scoped to the step
- Check execution is verified, not assumed: a PR showing zero checks is never reported green — when CI triggers skip stack-internal PRs, the documented local runs stand in as evidence and the gap is called out
- Every PR body is self-contained: stack position, parent branch, evidence, and merge order, with a living stack manifest kept current in PR #1 so async reviewers see the whole plan and progress from the bottom of the stack
- Runs autonomously when there's no one to confirm the breakdown with: proceeds with a best-effort plan recorded in PR #1 instead of blocking
- The repository's PR template is followed when one exists
- Changes to an earlier step are propagated forward through the chain, then re-verified
- Mid-run merges are handled: bases retargeted, remaining branches rebased (squash-merge aware), checks re-run
- Superseded PRs are closed with a one-line explanation, so the open stack only contains PRs worth a reviewer's attention
- Review feedback is triaged to the step that owns it: fixes land on the owning branch and propagate forward, and stack-artifact bot comments get a reply explaining the chain instead of a misplaced change
- The agent never merges PRs and never pushes to the default branch

Known stacked-PR gotchas (merge-queue checks that can't pass until parents merge, CI triggers that never fire on stack-internal PRs, review bots that skip drafts or lack stack awareness, squash-merge commit duplication) are documented with recovery commands in [`references/gotchas.md`](.agents/skills/cloud-stacked-diffs/references/gotchas.md). If you use Devin Review, [`references/devin-review.md`](.agents/skills/cloud-stacked-diffs/references/devin-review.md) has a copy-paste `REVIEW.md` snippet that teaches the bot your stack convention.

## Requirements

- git
- A way to create pull requests: a native Git/PR integration in your harness, or a platform CLI such as `gh` (GitHub) or `glab` (GitLab)

## Compatibility

Any agent that supports the open Agent Skills standard, including Devin (cloud and CLI), Claude Code, Codex, Cursor, OpenCode, and Zed.
