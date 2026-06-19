---
mode: agent
description: Push the current branch, open a PR, then dispatch the On-Demand Deploy + Smoke Test workflow with the branch name. (Stops before Azure Boards.)
---

# Open PR + Dispatch Deploy/Smoke Test

Automate my changes up to the pull request and CI run. Ask me for anything you
can't determine automatically, then run every step in order and stop on failure.

## Context to gather automatically

- **Repo**: from `git remote get-url origin` → owner/name.
- **Current branch**: `git rev-parse --abbrev-ref HEAD`.
- **Base branch**: the repo's default branch (usually `main`).

## Steps

1. **Validate local state**
   - Run `git status`.
   - If I'm on `main`/`master`, ask me for a new branch name and create it
     (`git checkout -b <name>`).
   - If there are uncommitted changes, ask for a commit message and commit them.

2. **Push the branch**
   - `git push -u origin <current-branch>`.

3. **Open the pull request**
   - Base: default branch. Head: current branch.
   - **Title**: concise summary derived from the commits (confirm with me).
   - **Body**: bullet summary of the changes.
   - Return the PR URL.

4. **Dispatch the On-Demand Deploy + Smoke Test workflow** (GitHub MCP server)
   - Call the MCP `run_workflow` tool for `deploy-smoke-test.yml`.
   - Dispatch `ref`: the default branch (`main`), where the workflow file lives.
   - Input `branch_name`: the current branch (the job checks out that branch).
   - Return the dispatched workflow run URL.

5. **Report** a short summary: branch, PR URL, and the deploy/smoke-test run URL.

> Note: no workflow runs automatically on PR open or branch push. The `CI`
> workflow runs **only when the PR is merged into `main`** (a merge is a push to
> `main`). The deploy/smoke-test workflow runs **only on demand** via this step.

