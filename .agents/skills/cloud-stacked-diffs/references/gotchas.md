# Stacked-PR gotchas and recovery

Field notes from running stacked-PR workflows with cloud agents. SKILL.md has the rules; this file has the why and the recovery commands.

## Verification is head-commit-specific

A green check "on the PR" is meaningless unless it ran against the branch's current head commit. Every push — including propagation merges and rebases — invalidates prior runs. After any change to a branch, re-run the repository's checks and re-check the PR's status before calling it verified. Citing a check that passed on an older commit is the single most common false done-claim in stacked workflows.

## Checks that cannot pass until parents merge

Merge-queue and stack-mergeability checks (Graphite's mergeability check, GitHub merge queues, base-branch protection checks) frequently stay pending or red on stacked PRs until every parent has merged. This is expected, not a failure:

- Do not idle waiting for them.
- Do not try to "fix" them by rebasing onto the default branch — that destroys the stack.
- Name them in the delivery summary so reviewers know which pending checks are merge-order artifacts.

## Review bots vs. drafts and stacks

Automated review bots (Devin Review, Cursor BugBot, CodeRabbit, Copilot review, ...) typically:

- Skip draft PRs entirely, and ignore trigger comments posted on drafts.
- May not re-run on stacked-base PRs after pushes or empty commits.

Reliable re-trigger recipe: mark the PR ready for review; if the bot still does not run, close and reopen it (`gh pr close <n> && gh pr reopen <n>`) — most bots auto-run on reopen against the current head. If the workflow requires PRs to stay drafts, note in the delivery summary that bot review will start when the reviewer flips them ready, bottom-up.

## Stack-unaware review bots

Per-PR review bots have no model of the chain: each PR is reviewed independently, so a bot can flag code that a parent step sets up or a child step already fixes. Three layers of defense:

1. **Geometry (automatic).** Because each PR's base is its parent branch, the parent's work is already in the diff context the bot reviews against. This alone prevents most false flags.
2. **Instruction.** If the repo uses Devin Review, add the stack convention to a root `REVIEW.md` so the bot reviews each PR against its base without demanding context that lives in the parent — copy-paste snippet in [devin-review.md](devin-review.md). Other bots have equivalents (Cursor BugBot reads `.cursor/BUGBOT.md`, CodeRabbit reads `.coderabbit.yaml`).
3. **Triage (the agent's job).** For residual flags, identify the step that owns the flagged code. Fix it there and propagate forward; never "fix" one step's concern inside another step's PR. If the flag is a stack artifact (provided by a parent, fixed by a child), reply on the thread explaining that so a reviewer can resolve it.

What no configuration can fix: cross-PR coordination of findings. The bot will never know that PR 4 resolves what it flagged in PR 3 — only triage absorbs that.

## Squash merges duplicate commits

If the repository squash-merges, the merged PR's individual commits never land on the default branch — so every remaining branch in the stack still carries the old step's commits and will conflict on a naive rebase or merge. Drop the duplicated range instead of replaying it:

```bash
git fetch origin
git rebase --onto origin/main feature/01-scaffold feature/02-design-tokens
git push --force-with-lease origin feature/02-design-tokens
```

Repeat up the chain (`--onto <new parent> <old parent> <branch>`), then re-run the repository's checks on every rebased branch.

## Base auto-retargeting

When a PR's base branch is deleted after merging, GitHub retargets that branch's child PRs to the merged PR's base automatically; other platforms may not, and the behavior differs when the branch is kept. Always verify each child PR's base after a merge (`gh pr view <n> --json baseRefName`) and retarget manually when needed (`gh pr edit <n> --base main`).

## Evidence hosting

Evidence must be viewable by the humans reviewing the PR:

- Upload screenshots and recordings directly in the PR body or a PR comment so the platform hosts them.
- Attachment or presigned links from an agent harness commonly 401 for anyone outside the session.
- If assets are hosted via a release, it must be published — draft releases 404 for everyone else.
- Never put private-product screenshots on public third-party hosts.
