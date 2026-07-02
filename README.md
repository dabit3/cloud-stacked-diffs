# stacked-cloud-diffs

Agent skill that turns big tasks into a reviewable stack of draft PRs. Works with Devin, Claude, Codex, and any harness supporting the [Agent Skills standard](https://agentskills.io/specification).

The skill is named `cloud-stacked-diffs` and lives at [`.agents/skills/cloud-stacked-diffs/SKILL.md`](.agents/skills/cloud-stacked-diffs/SKILL.md).

## Why

The most common failure mode of a cloud coding agent working async is delivering one giant, unreviewable PR. This skill teaches the agent to deliver multi-step work as a chain of stacked draft pull requests instead: one branch per step, each branching from and targeting the previous step's branch, so reviewers can approve the work in small increments and merge bottom-up.

## Install

With the [Skills CLI](https://skills.sh/):

```bash
npx skills add naderdabit/stacked-cloud-diffs@cloud-stacked-diffs
```

Or copy the skill directory into your project manually:

```bash
mkdir -p .agents/skills
cp -r path/to/stacked-cloud-diffs/.agents/skills/cloud-stacked-diffs .agents/skills/
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
 └─ feature/01-scaffold           PR #1 (base: main)
     └─ feature/02-design-tokens  PR #2 (base: feature/01-scaffold)
         └─ feature/03-app-shell  PR #3 (base: feature/02-design-tokens)
```

It finishes with a summary of the branch chain and all PR links, in merge order.

## What the skill enforces

- One branch and one draft PR per step, named `feature/NN-slug`
- Step 1 branches from the default branch; every later step branches from the previous step's branch
- Repository checks (lint, typecheck, tests) pass before each PR is opened
- The repository's PR template is followed when one exists
- Changes to an earlier step are propagated forward through the chain
- The agent never merges PRs and never pushes to the default branch

## Requirements

- git
- A way to create pull requests: a native Git/PR integration in your harness, or a platform CLI such as `gh` (GitHub) or `glab` (GitLab)

## Compatibility

Any agent that supports the open Agent Skills standard, including Devin (cloud and CLI), Claude Code, Codex, Cursor, OpenCode, and Zed.
