---
name: cloud-stacked-diffs
description: Deliver multi-step work as a chain of stacked draft pull requests — one branch per step, each branching from and targeting the previous step's branch. Use when the user asks for stacked PRs or stacked diffs, a PR stack or chain, incremental step-by-step review, or wants a large task split into small dependent PRs. Also use when updating, rebasing, or addressing review feedback on any PR that belongs to an existing stack (chained branches, e.g. stack/<name>/NN-* or feature/NN-*). Especially suited to cloud/background coding agents delivering large autonomous changes as reviewable increments.
compatibility: Requires git and a way to create pull requests (a native Git/PR integration, or a platform CLI such as gh or glab)
---

# Cloud Stacked Diffs

Deliver a multi-step task as an ordered chain of branches and draft pull requests, so each step can be reviewed independently and merged in order.

## Inputs

The ordered list of steps comes from the user's request. If the request does not already define clear steps, propose a numbered breakdown and confirm it with the user before starting. When running autonomously with no one to confirm with (a background/async agent), do not block: proceed with a best-effort breakdown, record it as the stack plan in the first PR's body, and refine it as the work reveals better boundaries.

Splitting into steps: each step should be one coherent concern that can pass the repository's checks on its own and be reviewed without reading the other steps. Prefer boundaries that keep a step self-contained over hitting any particular size. If a step cannot go green in isolation — e.g. its tests only pass once a later step lands — that is a bad split: reorder or combine steps rather than opening a red PR. Truly independent pieces of work do not need a stack at all; parallel PRs branched off the default branch are simpler.

## Rules (apply to every step)

- Before starting, create a todo list with one item per step. Use your built-in task tracking if you have it; otherwise maintain a markdown checklist.
- Pick ONE stack slug for the whole run: a short kebab-case name for the overall task (e.g. `checkout-flow`). All of the stack's branches live under `stack/<stack-slug>/`. Before first use, check the remote (`git ls-remote --heads origin "stack/<stack-slug>/*"`): existing branches under that prefix mean you are resuming that same task — if they belong to a different task, pick another slug. The namespace is what lets parallel stacks coexist in one repository.
- Name branches `stack/<stack-slug>/NN-step-slug`, where `NN` is the two-digit zero-padded step number and `step-slug` is a short kebab-case description of the step (e.g. `stack/checkout-flow/01-scaffold`, `stack/checkout-flow/02-design-tokens`).
- Step 1 branches from the repository's default branch. Detect it rather than assuming `main` (e.g. `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`, or `git symbolic-ref refs/remotes/origin/HEAD`). Command examples in this skill use `main` as a placeholder for whatever the default branch actually is.
- Every subsequent step branches from the PREVIOUS step's branch, never from the default branch.
- Every PR is created as a draft.
- Keep each step's diff scoped to that step only.
- Verification is head-commit-specific: a check that passed on an older commit means nothing after a push. Any change to a branch — fixes, propagation merges, rebases — requires re-running the repository's checks on it. Zero checks executing is not a pass either: confirm checks actually ran before reporting a PR green.
- Keep a living stack manifest in PR #1's body: the full numbered plan plus each step's status and PR link, updated as steps land. It is the async source of truth so a reviewer landing on the bottom of the stack sees the whole plan and progress. The end-of-run overview comment (see Delivery) is its final snapshot.
- Long runs drift: re-read this skill's rules before starting each step.

## Procedure (repeat for each step N)

1. Pick a short, descriptive slug for the step.
2. Create the branch from the correct base: the previous step's branch, or the default branch for step 1. If the branch already exists (e.g. you are resuming an interrupted run), verify it belongs to this stack before building on it — it lives under this stack's `stack/<stack-slug>/` prefix and its PR, if one exists, targets the expected parent — then check it out and continue where it left off. Never resume onto a branch that fails that check: it is another task's work, and the fix is a different stack slug, not a shared branch.
3. Implement the work for the step on that branch.
4. Run the repository's configured checks (lint, typecheck, and tests if present). Find them where the repo defines them — `package.json` scripts, a `Makefile`/`Justfile`, `pre-commit` config, `pyproject.toml`/`tox.ini`, or the CI workflow files (which also show what platform CI will run). If you genuinely cannot find any checks, do not silently treat that as green — say so in the PR body. If any fail, fix them on this branch before moving on.
5. Commit and push the branch to the remote.
6. Create a draft pull request:
   - head: this step's branch
   - base: the previous step's branch (the default branch for step 1)
   - Follow the repository's PR template if one exists (check `.github/PULL_REQUEST_TEMPLATE.md` and the other standard template locations).
   - Make the body self-contained — reviewers land on individual PRs without context: state the step's position ("Step N of M"), its parent branch, that the diff shows only this step because the base is the previous step's branch, and that the stack merges bottom-up.
   - Include evidence in the body: which checks ran and their results. For UI work, upload screenshots or recordings to the PR itself — attachment links hosted by your own harness are often not viewable by humans.
   - Use whatever PR mechanism is available: your harness's native Git/PR integration if it has one, otherwise the platform CLI (e.g. `gh pr create --draft --head <branch> --base <base>` for GitHub, `glab mr create --draft` for GitLab).
7. Verify the wiring — do not trust the create call. Confirm the PR's base is the intended parent branch (e.g. `gh pr view <n> --json baseRefName,headRefName`) and that its changed files contain only this step's work. A stacked PR accidentally based on the default branch shows the whole cumulative diff; fix with `gh pr edit <n> --base <parent>`.
8. Verify checks executed — CI triggers are often filtered to PRs targeting the default branch, so a stack-internal PR can show zero checks. Confirm platform CI actually ran against the head commit (e.g. `gh pr checks <n>`). If it never triggers for stack-based PRs, "no failures" is not green: state the gap in the PR body and cite the step 4 local runs (commands and results) as the evidence. See [references/gotchas.md](references/gotchas.md).
9. Mark the step's todo item complete, then move to step N+1.

## If an earlier step needs changes

The branches are chained, so changing step K affects every step after it:

1. Make the fix on step K's branch and push it.
2. Propagate forward in order: for each following branch (K+1, then K+2, ...), merge its parent branch into it, resolve any conflicts, and push. Resolve clerical conflicts (import ordering, adjacent edits, lockfiles) yourself; if a conflict is substantive or its intended resolution is ambiguous, stop and escalate rather than guessing.
3. If the team prefers linear history, instead rebase each descendant onto its updated parent and push with `--force-with-lease` — but only ever force-push the stack's own `stack/<stack-slug>/*` branches.
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
- Platform CI may not trigger at all on stack-internal PRs: workflow triggers commonly filter to the default branch, so a stacked PR can show zero checks. Absence of checks is not success — verify execution per procedure step 8 and name affected PRs in the delivery summary.
- Automated review bots typically skip draft PRs, and on stacked PRs a push or empty commit may not re-trigger them (closing and reopening the PR usually does). Never report a bot or check as green unless it ran against the branch's current head commit.
- Review bots have no concept of the chain: they review each PR independently and may flag things a parent sets up or a child already fixes. Triage per "Handling review feedback on the stack"; if the repo uses Devin Review, add the stack convention to a root `REVIEW.md` (see [references/devin-review.md](references/devin-review.md)).
- Details and recovery commands: [references/gotchas.md](references/gotchas.md).

## Delivery

After ALL steps are done:

1. Sweep every PR in the stack: still a draft, base points at its parent branch, checks executed and green on the current head commit (a PR with no checks at all is not green — apply procedure step 8; any failure explained), evidence present.
2. Post (or refresh) a stack-overview comment on every PR listing all PRs in merge order, marking that PR's position.
3. Send a single summary message with:
   - The branch chain: `main → stack/<slug>/01-... → stack/<slug>/02-... → ... → stack/<slug>/NN-...`
   - A numbered list of all PR links, in stack order
   - A note that the PRs should be reviewed and merged bottom-up (step 1 first)
   - Any checks that can only pass after parents merge, and any PRs where platform CI never triggered (with the local check runs that stand in as evidence)

## Forbidden

- Do NOT merge any PR.
- Do NOT push directly to the default branch.
- Do NOT base a step's branch on anything other than the previous step's branch (or the default branch for step 1).
- Do NOT delete a stack branch while a later step's PR still targets it as its base.
