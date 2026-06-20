---
name: create-pull-request
description: >-
  Push the current branch, open a GitHub pull request tagged with an Azure Work
  Item number (AB#), and dispatch the On-Demand Deploy + Smoke Test workflow.
  Use when the user wants to create or open a pull request, raise/submit a PR,
  ship the current changes, or kick off deploy + smoke test for a branch. Handles
  branch validation, optional commit and squash, rebasing onto the default
  branch, building the PR body from the repository PR template, and dispatching
  the workflow. Triggers on phrases like "open a PR", "create a pull request",
  "raise a PR for my changes", or "deploy and smoke test my branch".
---

# Create Pull Request

Create a pull request for the current changes and kick off the on-demand
**Deployment + Smoke Test** workflow. Ask the user for anything that cannot be
determined automatically, then run every step in order and **stop on failure**.

## Context to gather automatically

- **Repo**: from `git remote get-url origin` → `owner/name`.
- **Current branch**: `git rev-parse --abbrev-ref HEAD`.
- **Base branch**: the repo's default branch (usually `main`).
- **Azure Work Item**: `AB#{work_item_number}` (ask the user if not provided).

## Steps

1. **Gather the Azure Work Item number**
   - Ask the user for the Azure Work Item number (`AB#{number}`) if it is not
     mentioned in the commit messages or branch name.
   - Store it for the PR title and the final report.

2. **Validate local state**
   - Run `git status`.
   - If on `main`/`master`, ask the user for a new branch name and create it
     (`git checkout -b <name>`).
   - **If there are uncommitted changes, check with the user whether to commit them or not.**
     - Ask the user to choose one: **commit**, **continue without committing**, or **quit**.
     - If the user chooses **commit**, ask for a commit message and commit them.
     - If the user chooses **continue**, leave the changes uncommitted and proceed
       using only the already-committed work.
     - If the user chooses **quit**, stop the workflow here and report it was cancelled.

3. **Squash multiple commits (only when safe)**
   - **First, check whether the current branch is up to date with the default branch.**
     - Fetch the latest base: `git fetch origin <base-branch>`.
     - Count base commits missing from the branch:
       `git rev-list --count HEAD..origin/<base-branch>`.
   - **If the branch is NOT up to date** (the default branch has commits the branch lacks):
     - **Do NOT squash.**
     - Update the branch with the latest base instead
       (`git merge origin/<base-branch>` or `git rebase origin/<base-branch>`),
       then proceed without squashing.
   - **If the branch IS up to date** with the default branch:
     - Check whether there are multiple commits on the branch compared to base.
     - If multiple commits exist, squash them into one with a meaningful message
       derived from the commit history.
     - Otherwise, proceed without squashing.

4. **Push the branch**
   - Check whether the branch has local commits that aren't on the remote yet
     (e.g., `git status` shows "ahead", or
     `git rev-list --count origin/<current-branch>..HEAD` > 0).
   - **If there are commits to push**: `git push -u origin <current-branch>`
     (or `git push -f origin <current-branch>` if the branch was rebased).
   - **If there are NO commits to push** (nothing new and the branch is already pushed):
     - Check whether a pull request already exists for this branch.
     - **If no PR exists** for the branch, continue to step 5 and create one anyway.
     - **If a PR already exists**, return its URL and skip creating a duplicate.

5. **Open the pull request**
   - Base: default branch. Head: current branch.
   - **Title**: include the Azure Work Item number —
     `AB#{work_item} | <concise summary>` derived from the commits.
   - **Body**: follow the repository PR template structure (see
     [Pull request body structure](#pull-request-body-structure) below, sourced
     from `.github/pull_request_template.md`).
   - Create it with the **GitHub MCP server** (`create_pull_request`) **or** the
     `gh` CLI:
     ```bash
     gh pr create --base <base-branch> --head <current-branch> \
       --title "AB#<work_item> | <summary>" --body-file <body-file>
     ```
   - Return the PR URL.

6. **Dispatch the On-Demand Deploy + Smoke Test workflow**
   - Workflow file: `deploy-smoke-test.yml`
     (jobs `deploy` → `smoke-test`, triggered by `workflow_dispatch`).
   - Dispatch `ref`: the **default branch** (`main`), where the workflow file lives.
   - Input `branch_name`: the **current branch** (both jobs check out this branch).
   - Use the GitHub MCP `run_workflow` tool **or** the `gh` CLI:
     ```bash
     gh workflow run deploy-smoke-test.yml --ref main -f branch_name=<current-branch>
     ```
   - Return the dispatched workflow run URL.

7. **Report** a short summary: branch, Azure Work Item (`AB#`), PR URL, and the
   deployment/smoke-test run URL.

## Pull request body structure

Mirror `.github/pull_request_template.md`:

```markdown
### Platform
##### Pull request information

###### Quality Checks
- What is the feature or problem that this PR address?

- What has been done in the source code to address this?

- How did you test it?

- Any other relevant information to reviewer?

- Backend/Frontend PR Link if any:

#### AMPM Platform - Definition of Done (DoD) - Reviewer checklist
##### As a reviewer I have checked _all_ the items mentioned below:

- [ ] All the gated checks are passing
- [ ] The code has been reviewed observing the business requirements and best practices
- [ ] The code has propper code abstraction
- [ ] No microcode duplication has been found in this pull request
- [ ] Any By-pass for this PR? If Yes, please provide the details here - Failure and Rationale
```

Fill in the **Quality Checks** answers from the actual changes; leave the DoD
checklist unchecked for the reviewer.

## Guardrails

- Run steps **in order** and **stop on the first failure**, reporting what failed.
- Never force-push unless the branch was rebased in step 3.
- Do not squash when the branch is behind the default branch.
- Always include the `AB#` work item number in the PR title.
- When there are uncommitted changes, always **check with the user** whether to
  commit them, continue without committing, or quit — never auto-commit silently.
- Never create a duplicate PR: if a PR already exists for the branch, return it
  instead. If the branch is already pushed with nothing new to push and no PR
  exists, still create the PR.
