---
name: create-pull-request
description: >-
  Push the current branch, open a GitHub pull request tagged with an Azure Work
  Item number (AB#), and dispatch the On-Demand Deploy + Smoke Test workflow.
  Use when the user wants to create or open a pull request, raise/submit a PR,
  ship the current changes, or kick off deploy + smoke test for a branch. Handles
  branch validation, optional commit, a read-only sync check against the default
  branch (never auto-updating), production-safe squashing, building the PR body
  from the repository PR template, and dispatching the workflow. Triggers on
  phrases like "open a PR", "create a pull request", "raise a PR for my changes",
  or "deploy and smoke test my branch".
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

1. **Check sync with the default branch FIRST (read-only — quit if not in sync)**
   - This is the **first gate**. Run it before anything else.
   - Fetch the base ref only — this does **not** modify the working branch:
     `git fetch origin <base-branch>`.
   - Count how many base commits the branch is missing:
     `git rev-list --count HEAD..origin/<base-branch>`.
   - **If the count is 0** (branch is in sync), continue to the next step.
   - **If the branch is behind** (count > 0): **simply quit.**
     - **Do NOT run `git merge`, `git rebase`, or `git pull` on the user's behalf** —
       never update the branch automatically.
     - Report that the branch is behind `origin/<base-branch>` by N commits and that the
       user must sync it with the default branch themselves, then re-run this workflow.
     - Stop here.

2. **Gather the Azure Work Item number**
   - Ask the user for the Azure Work Item number (`AB#{number}`) if it is not
     mentioned in the commit messages or branch name.
   - Store it for the PR title and the final report.

3. **Validate local state**
   - Run `git status`.
   - If on `main`/`master`, ask the user for a new branch name and create it
     (`git checkout -b <name>`).
   - **If there are uncommitted changes (staged, unstaged, or untracked), check with the
     user whether to commit them.** Treat phrasings like "commit", "commit and continue",
     or "commit them" as the **commit** choice.
     - Ask the user to choose one: **commit**, **continue without committing**, or **quit**.
     - If the user chooses **commit**, you MUST actually create the commit — do not just
       acknowledge it or move on. Run these steps in order:
       1. Ask the user for a commit message (or propose one from the changes and confirm).
       2. Stage everything: `git add -A`.
       3. Create the commit: `git commit -m "<message>"`.
       4. Verify it worked: run `git status` (the working tree should now be clean) and
          `git log -1 --oneline` to confirm the new commit exists.
       5. Only continue once the commit is confirmed. If nothing was staged or the commit
          failed/was empty, tell the user and stop — never silently continue.
     - If the user chooses **continue without committing**, leave the changes uncommitted
       and proceed using only the already-committed work.
     - If the user chooses **quit**, stop the workflow here and report it was cancelled.

4. **Squash commits (production-safe — only when it cannot corrupt the branch)**
   - **Preferred strategy: do NOT rewrite local history.** Keep the commits as they are
     and rely on GitHub's **"Squash and merge"** option at merge time. This is the safest
     production approach because it never touches the working branch and is safe even for
     shared branches.
   - Only consider squashing locally when **all** of these hold:
     - The branch has multiple small/WIP commits ("wip", "fix typo", "address review")
       that add no historical value.
     - **Every commit is local-only and unshared** — verify with
       `git rev-list --count origin/<current-branch>..HEAD` and confirm no one else works
       on the branch.
     - The branch is **up to date** with the default branch (confirmed in step 1).
   - **Never squash / rewrite history when**:
     - The branch is already pushed or shared (rewriting forces a force-push that can
       corrupt teammates' work).
     - The branch is behind the default branch.
     - There is only one commit, or the commits are already meaningful and atomic.
   - If a local squash is justified, first take a safety backup of the current tip
     (`git branch <current-branch>-backup`), then squash with
     `git reset --soft <merge-base>` (or `git rebase -i <merge-base>`) using one
     meaningful message derived from the history.
   - **When in doubt, skip local squashing** and let the PR be squash-merged.

5. **Push the branch**
   - Check whether the branch has local commits that aren't on the remote yet
     (e.g., `git status` shows "ahead", or
     `git rev-list --count origin/<current-branch>..HEAD` > 0).
   - **If there are commits to push**: `git push -u origin <current-branch>`.
     - Use `git push -f` only if already-pushed history was explicitly rewritten in
       step 4 (normally avoided); otherwise never force-push.
   - **If there are NO commits to push** (nothing new and the branch is already pushed):
     - Check whether a pull request already exists for this branch.
     - **If no PR exists** for the branch, continue to the next step and create one anyway.
     - **If a PR already exists**, return its URL and skip creating a duplicate.

6. **Open the pull request**
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

7. **Dispatch the On-Demand Deploy + Smoke Test workflow**
   - Workflow file: `deploy-smoke-test.yml`
     (jobs `deploy` → `smoke-test`, triggered by `workflow_dispatch`).
   - Dispatch `ref`: the **default branch** (`main`), where the workflow file lives.
   - Input `branch_name`: the **current branch** (both jobs check out this branch).
   - Use the GitHub MCP `run_workflow` tool **or** the `gh` CLI:
     ```bash
     gh workflow run deploy-smoke-test.yml --ref main -f branch_name=<current-branch>
     ```
   - Return the dispatched workflow run URL.

8. **Report** a short summary: branch, Azure Work Item (`AB#`), PR URL, and the
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
- **Never update the working branch automatically.** Do not run `git merge`,
  `git rebase`, or `git pull` against the default branch — if the branch is behind,
  **simply quit** and tell the user to sync it with the default branch themselves,
  then re-run.
- **Production-safe squashing only.** Prefer GitHub "Squash and merge" at merge time;
  only squash locally when every commit is local-only/unshared and the branch is up to
  date, and take a `<branch>-backup` first. Never rewrite shared or already-pushed
  history, and never squash a branch that is behind the default branch.
- Never force-push unless already-pushed history was explicitly rewritten in step 4
  (normally avoided).
- Always include the `AB#` work item number in the PR title.
- When there are uncommitted changes, always **check with the user** whether to
  commit them, continue without committing, or quit — never auto-commit silently.
  When the user chooses to commit, you must **actually run `git add -A` then
  `git commit -m "<message>"` and verify** with `git status`/`git log -1` — never
  just acknowledge the choice without creating the commit.
- Never create a duplicate PR: if a PR already exists for the branch, return it
  instead. If the branch is already pushed with nothing new to push and no PR
  exists, still create the PR.

