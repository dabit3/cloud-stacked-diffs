---
name: cloud-stacked-diffs
description: Deliver multi-step work as a chain of stacked draft pull requests — one branch per step, each branching from and targeting the previous step's branch. Use when the user asks for stacked PRs or stacked diffs, a PR stack or chain, incremental step-by-step review, or wants a large task split into small dependent PRs. Also use when updating, rebasing, or addressing review feedback on any PR that belongs to an existing stack (chained branches, e.g. feature/NN-*). Especially suited to cloud/background coding agents delivering large autonomous changes as reviewable increments.
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
- Verification is head-commit-specific: a check that passed on an older commit means nothing after a push. Any change to a branch — fixes, propagation merges, rebases — requires re-running the repository's checks on it.
- Long runs drift: re-read this skill's rules before starting each step.

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
   - Make the body self-contained — reviewers land on individual PRs without context: state the step's position ("Step N of M"), its parent branch, that the diff shows only this step because the base is the previous step's branch, and that the stack merges bottom-up.
   - Include evidence in the body: which checks ran and their results. For UI work, upload screenshots or recordings to the PR itself — attachment links hosted by your own harness are often not viewable by humans.
   - Use whatever PR mechanism is available: your harness's native Git/PR integration if it has one, otherwise the platform CLI (e.g. `gh pr create --draft --head <branch> --base <base>` for GitHub, `glab mr create --draft` for GitLab).
7. Verify the wiring — do not trust the create call. Confirm the PR's base is the intended parent branch (e.g. `gh pr view <n> --json baseRefName,headRefName`) and that its changed files contain only this step's work. A stacked PR accidentally based on the default branch shows the whole cumulative diff; fix with `gh pr edit <n> --base <parent>`.
8. Mark the step's todo item complete, then move to step N+1.

## If an earlier step needs changes

The branches are chained, so changing step K affects every step after it:

1. Make the fix on step K's branch and push it.
2. Propagate forward in order: for each following branch (K+1, then K+2, ...), merge its parent branch into it, resolve any conflicts, and push.
3. If the team prefers linear history, instead rebase each descendant onto its updated parent and push with `--force-with-lease` — but only ever force-push the stack's own `feature/NN-*` branches.
4. Re-verify everything you touched: re-run the repository's checks on every propagated branch. A merge or rebase can break a descendant even when there were no conflicts.
5. If a step is dropped or reworked to the point its PR is obsolete, close that PR with a one-line comment ("superseded by #N"). The open stack must only contain PRs worth a reviewer's attention.

## Handling review feedback on the stack

When asked to address review comments (human or bot) on any PR in the stack:

1. Enumerate all unresolved review threads across EVERY PR in the chain, not just the PR named in the request (`gh pr view <n> --comments`, or the GraphQL `reviewThreads` API with `isResolved`).
2. Classify each comment by which step OWNS the flagged code. Fix it on that step's branch, then propagate forward per "If an earlier step needs changes".
3. If a comment on PR N is owned by a different step (a parent sets it up, or a child already fixes it), do not duplicate a change into PR N — that breaks step scoping. Fix the owning step, or reply on the thread explaining the stack context so a reviewer can resolve it.

## If a step merges mid-run

If you resume work and find the bottom of the stack already merged:

1. Confirm the next unmerged PR now targets the default branch. Some platforms retarget child PRs automatically when the merged branch is deleted — verify, never assume, and retarget manually if needed (`gh pr edit <n> --base main`).
2. Rebase the remaining chain bottom-up onto the new base. If the repository squash-merges, the merged step's commits are duplicated in your branches and will conflict on a naive rebase — drop them with `git rebase --onto <new-base> <old-parent> <branch>`, then push with `--force-with-lease`.
3. Re-run the repository's checks on every rebased branch, per the re-verification rule.

## Stacked-PR gotchas

- Merge-queue and stack-mergeability checks on a stacked PR often cannot pass until its parents merge. Never idle waiting on one — note it in the delivery summary instead.
- Automated review bots typically skip draft PRs, and on stacked PRs a push or empty commit may not re-trigger them (closing and reopening the PR usually does). Never report a bot or check as green unless it ran against the branch's current head commit.
- Review bots have no concept of the chain: they review each PR independently and may flag things a parent sets up or a child already fixes. Triage per "Handling review feedback on the stack"; if the repo uses Devin Review, add the stack convention to a root `REVIEW.md` (see [references/devin-review.md](references/devin-review.md)).
- Details and recovery commands: [references/gotchas.md](references/gotchas.md).

## Delivery

After ALL steps are done:

1. Sweep every PR in the stack: still a draft, base points at its parent branch, checks green on the current head commit (or the failure explained), evidence present.
2. Post (or refresh) a stack-overview comment on every PR listing all PRs in merge order, marking that PR's position.
3. Send a single summary message with:
   - The branch chain: `main → feature/01-... → feature/02-... → ... → feature/NN-...`
   - A numbered list of all PR links, in stack order
   - A note that the PRs should be reviewed and merged bottom-up (step 1 first)
   - Any checks that can only pass after parents merge

## Forbidden

- Do NOT merge any PR.
- Do NOT push directly to the default branch.
- Do NOT base a step's branch on anything other than the previous step's branch (or the default branch for step 1).
