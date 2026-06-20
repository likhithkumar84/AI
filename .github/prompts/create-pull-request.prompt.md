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

1. **Check sync with the default branch FIRST (read-only — quit if not in sync)**
    - This is the **first gate**. Run it before anything else.
    - Fetch the base ref only (this does **not** change my working branch):
      `git fetch origin <base-branch>`.
    - Count how many base commits my branch is missing:
      `git rev-list --count HEAD..origin/<base-branch>`.
    - **If the count is 0** (branch is in sync), continue to the next step.
    - **If my branch is behind** (count > 0): **simply quit.**
      - **Do NOT run any `git merge`, `git rebase`, or `git pull` yourself** — never
        update the branch automatically.
      - Report that the branch is behind `origin/<base-branch>` by N commits and that I
        must sync it with the default branch myself, then re-run this workflow.
      - Stop the workflow here.

2. **Gather Azure Work Item number**
    - Ask me for the Azure Work Item number (AB#{number}) if it's not mentioned in the commit messages or branch name.
    - Store this for use in the PR title and final report.

3. **Validate local state**
    - Run `git status`.
    - If I'm on `main`/`master`, ask me for a new branch name and create it
      (`git checkout -b <name>`).
    - **If there are uncommitted changes (staged, unstaged, or untracked), check with me
      whether to commit them.** Treat phrasings like "commit", "commit and continue", or
      "commit them" as the **commit** choice.
      - Ask me to choose one: **commit**, **continue without committing**, or **quit**.
      - If I choose **commit**, you MUST actually create the commit — do not just
        acknowledge it or move on. Run these steps in order:
        1. Ask me for a commit message (or propose one from the changes and confirm it).
        2. Stage everything: `git add -A`.
        3. Create the commit: `git commit -m "<message>"`.
        4. Verify it worked: run `git status` (the working tree should now be clean) and
           `git log -1 --oneline` to confirm the new commit exists.
        5. Only continue once the commit is confirmed. If nothing was staged or the
           commit failed/was empty, tell me and stop — never silently continue.
      - If I choose **continue without committing**, leave the changes uncommitted and proceed using only the already-committed work.
      - If I choose **quit**, stop the workflow here and report that it was canceled.

4. **Squash commits (production-safe — only when it cannot corrupt the branch)**
    - **Preferred strategy: do NOT rewrite local history.** Keep the commits as they
      are and rely on GitHub's **"Squash and merge"** option at merge time. This is the
      safest production approach because it never touches the working branch and works
      even for shared branches.
    - Only consider squashing locally when **all** of these are true:
      - The branch has multiple small/WIP commits ("wip", "fix typo", "address review")
        that add no historical value.
      - **Every commit is local-only and unshared** — verify with
        `git rev-list --count origin/<current-branch>..HEAD` and confirm no one else is
        working on the branch.
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
      (e.g., `git status` shows "ahead", or `git rev-list --count origin/<current-branch>..HEAD` > 0).
    - **If there are commits to push**: `git push -u origin <current-branch>`.
      - Only use `git push -f` if I explicitly rewrote already-pushed history in step 4
        (normally avoided); otherwise never force-push.
    - **If there are NO commits to push** (nothing new and the branch is already pushed):
      - Check whether a pull request already exists for this branch.
      - **If no PR exists** for the branch, continue to the next step and create one anyway.
      - **If a PR already exists**, return its URL and skip creating a duplicate.

6. **Open the pull request** (GitHub MCP server)
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

7. **Dispatch the On-Demand Deploy + Smoke Test workflow** (GitHub MCP server → Actions)
    - Call the MCP `run_workflow` tool for `deploy-smoke-test.yml`.
    - Dispatch `ref`: the default branch (`main`), where the workflow file lives.
    - Input `branch_name`: the current branch (the `deploy` and `smoke-test`
      jobs both check out that branch).
    - Return the dispatched workflow run URL.

8. **Report** a short summary: branch, Azure Work Item (AB#), PR URL, and the deployment/smoke-test run URL.
