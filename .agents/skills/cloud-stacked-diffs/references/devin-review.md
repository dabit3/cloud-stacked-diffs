# Devin Review setup for stacked PRs

Devin Review reads instruction files named `REVIEW.md` to customize how it reviews PRs (it picks up `**/REVIEW.md` at any directory level; see the [Devin Review docs](https://docs.devin.ai/work-with-devin/devin-review)). Review bots have no built-in concept of a PR stack, so without guidance they may flag code in step N that step N-1 provides.

Add the following to the `REVIEW.md` at the repository root, creating the file if it does not exist. Keep it at the root: a `REVIEW.md` nested in a subdirectory scopes its guidance to that subdirectory only.

```markdown
## Stacked pull requests

PRs on branches named `feature/NN-*` are stacked: each PR's base is the
previous step's branch, and the stack merges bottom-up (step 01 first).

When reviewing these PRs:

- Review only the diff against the PR's base branch.
- Do not flag missing context, setup, or definitions that exist in the
  parent branch (the base).
- A step may intentionally depend on unmerged parent PRs; that is not
  an error.
- Do not suggest retargeting the PR to the default branch.
```

Other review bots have similar mechanisms — Cursor BugBot reads `.cursor/BUGBOT.md`, CodeRabbit reads `.coderabbit.yaml` — so the same guidance can be adapted to their formats.

Note the limits: instruction files shape the bot's judgment within a single PR's review, but no configuration makes a per-PR bot coordinate findings across the chain (for example, knowing that a child PR fixes something it flagged in a parent). Those residual cases are handled by the triage rule in SKILL.md's "Handling review feedback on the stack".
