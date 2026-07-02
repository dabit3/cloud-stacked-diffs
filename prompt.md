## Stacked PR Rules (apply to ALL steps below)

Before starting, create a todo list with one item per step.

For each step N (starting at 1):
1. Determine a short, descriptive slug for the step (e.g. "scaffold", "design-tokens", "app-shell")
2. Create branch: `feature/{N:02d}-{slug}` (e.g. `feature/01-scaffold`, `feature/02-design-tokens`)
      - Step 1 branches from `main`
      - Every subsequent step branches from the PREVIOUS step's branch
3. Implement the work for that step on that branch
4. Run lint/typecheck before creating the PR
5. Call `fetch_pr_template` for this repo
6. Call `git_create_pr` with:
      - `head_branch`: the current step's branch
      - `base_branch`: the PREVIOUS step's branch (or `main` for step 1)
      - `draft`: true
7. Mark the todo item complete, then move to step N+1

After ALL steps are done, send a summary message with:
- The branch chain (step 1 → step 2 → ... → step N)
- A numbered list of all PR links
- Do NOT merge any PR
