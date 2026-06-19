---
mode: agent
description: Push the current branch, open a pull request with Azure Work Item number, then dispatch the On-Demand Deploy + Smoke Test workflow with the branch name.
---

# Create Pull Request

Create a pull request for my current changes and kick off the on-demand
**Deployment + Smoke Test** workflow. Ask me for anything you can't determine
automatically, then run every step in order and stop on failure.

## Context to gather automatically

- **Repo**: from `git remote get-url origin` → owner/name.
- **Current branch**: `git rev-parse --abbrev-ref HEAD`.
- **Base branch**: the repo's default branch (usually `main`).
- **Azure Work Item**: AB#{work_item_number} (ask me if not provided).

## Steps

1. **Gather Azure Work Item number**
    - Ask me for the Azure Work Item number (AB#{number}) if it's not mentioned in the commit messages or branch name.
    - Store this for use in the PR title and final report.

2. **Validate local state**
    - Run `git status`.
    - If I'm on `main`/`master`, ask me for a new branch name and create it
      (`git checkout -b <name>`).
    - If there are uncommitted changes, ask for a commit message and commit them.

3. **Squash multiple commits (if applicable)**
    - **First, check whether the current branch is up to date with the default branch.**
      - Fetch the latest base: `git fetch origin <base-branch>`.
      - Count base commits missing from the branch: `git rev-list --count HEAD..origin/<base-branch>`.
    - **If the branch is NOT up to date** (the default branch has commits the branch doesn't have):
      - **Do NOT squash.**
      - Update the branch with the latest base instead (e.g., `git merge origin/<base-branch>` or `git rebase origin/<base-branch>`), then proceed without squashing.
    - **If the branch IS up to date** with the default branch:
      - Check if there are multiple commits on the current branch compared to the base branch.
      - If multiple commits exist, squash all commits into one with a meaningful message (derived from commit history).
      - If no, proceed without squashing.

4. **Push the branch**
    - `git push -u origin <current-branch>` (or `git push -f origin <current-branch>` if rebased).

5. **Open the pull request** (GitHub MCP server)
    - Base: default branch. Head: current branch.
    - **Title**: Include Azure Work Item number — format: `AB#{work_item} | <concise summary>` derived from commits.
    - **Body**: Follow the PR template structure (from `.github/pull_request_template.md`):
      - Platform section header
      - Pull request information → Quality Checks:
        - What is the feature or problem that this PR addresses?
        - What has been done in the source code to address this?
        - How did you test it?
        - Any other relevant information to reviewer?
        - Backend/Frontend PR Link if any:
      - Include the AMPM Platform - Definition of Done (DoD) - Reviewer checklist
    - Return the PR URL.

6. **Dispatch the On-Demand Deploy + Smoke Test workflow** (GitHub MCP server → Actions)
    - Call the MCP `run_workflow` tool for `deploy-smoke-test.yml`.
    - Dispatch `ref`: the default branch (`main`), where the workflow file lives.
    - Input `branch_name`: the current branch (the `deploy` and `smoke-test`
      jobs both check out that branch).
    - Return the dispatched workflow run URL.

7. **Report** a short summary: branch, Azure Work Item (AB#), PR URL, and the deployment/smoke-test run URL.
