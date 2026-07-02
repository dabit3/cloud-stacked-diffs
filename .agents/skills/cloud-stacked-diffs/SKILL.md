---
name: cloud-stacked-diffs
description: Deliver multi-step work as a chain of stacked draft pull requests — one branch per step, each branching from and targeting the previous step's branch. Use when the user asks for stacked PRs or stacked diffs, a PR stack or chain, incremental step-by-step review, or wants a large task split into small dependent PRs. Especially suited to cloud/background coding agents delivering large autonomous changes as reviewable increments.
compatibility: Requires git and a way to create pull requests (a native Git/PR integration, or a platform CLI such as gh or glab)
---

# Cloud Stacked Diffs

Deliver a multi-step task as an ordered chain of branches and draft pull requests, so each step can be reviewed independently and merged in order.

## Inputs

The ordered list of steps comes from the user's request. If the request does not already define clear steps, propose a numbered breakdown (one reviewable unit of work per step) and confirm it with the user before starting.

## Rules (apply to every step)

- Before starting, create a todo list with one item per step. Use your built-in task tracking if you have it; otherwise maintain a markdown checklist.
- Name branches `feature/NN-slug`, where `NN` is the two-digit zero-padded step number and `slug` is a short kebab-case description of the step (e.g. `feature/01-scaffold`, `feature/02-design-tokens`).
- Step 1 branches from the repository's default branch (`main` unless the repo uses something else).
- Every subsequent step branches from the PREVIOUS step's branch, never from the default branch.
- Every PR is created as a draft.
- Keep each step's diff scoped to that step only.

## Procedure (repeat for each step N)

1. Pick a short, descriptive slug for the step.
2. Create the branch from the correct base: the previous step's branch, or the default branch for step 1. If the branch already exists (e.g. you are resuming an interrupted run), check it out and continue where it left off.
3. Implement the work for the step on that branch.
4. Run the repository's configured checks (lint, typecheck, and tests if present). If any fail, fix them on this branch before moving on.
5. Commit and push the branch to the remote.
6. Create a draft pull request:
   - head: this step's branch
   - base: the previous step's branch (the default branch for step 1)
   - Follow the repository's PR template if one exists (check `.github/PULL_REQUEST_TEMPLATE.md` and the other standard template locations).
   - Use whatever PR mechanism is available: your harness's native Git/PR integration if it has one, otherwise the platform CLI (e.g. `gh pr create --draft --head <branch> --base <base>` for GitHub, `glab mr create --draft` for GitLab).
7. Mark the step's todo item complete, then move to step N+1.

## If an earlier step needs changes

The branches are chained, so changing step K affects every step after it:

1. Make the fix on step K's branch and push it.
2. Propagate forward in order: for each following branch (K+1, then K+2, ...), merge its parent branch into it, resolve any conflicts, and push.
3. If the team prefers linear history, instead rebase each descendant onto its updated parent and push with `--force-with-lease` — but only ever force-push the stack's own `feature/NN-*` branches.

## Delivery

After ALL steps are done, send a single summary message with:

- The branch chain: `main → feature/01-... → feature/02-... → ... → feature/NN-...`
- A numbered list of all PR links, in stack order
- A note that the PRs should be reviewed and merged bottom-up (step 1 first)

## Forbidden

- Do NOT merge any PR.
- Do NOT push directly to the default branch.
- Do NOT base a step's branch on anything other than the previous step's branch (or the default branch for step 1).
