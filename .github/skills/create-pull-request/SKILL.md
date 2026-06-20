---
name: create-pull-request
description: >-
  Open a PR for the current branch, include an Azure Work Item number (AB#),
  and dispatch the On-Demand Deploy + Smoke Test workflow. Use for requests
  like "open a PR", "create a pull request", or "deploy and smoke test my branch".
---

# Create Pull Request

Create a PR for the current branch and dispatch the on-demand deployment + smoke test workflow. Ask only for missing information, run steps in order, and **stop on first failure**.

## Gather automatically

- repo from `git remote get-url origin`
- current branch from `git rev-parse --abbrev-ref HEAD`
- default branch from the remote
- `AB#{number}` from branch/commits, otherwise ask the user

## Steps

1. **Sync gate first**
   - Run `git fetch origin <base-branch>` and `git rev-list --count HEAD..origin/<base-branch>`.
   - If the count is greater than 0, stop and tell the user to sync with the default branch themselves.
   - Never run `git merge`, `git rebase`, or `git pull` on the user's behalf.

2. **Get the work item**
   - If `AB#...` cannot be inferred, ask the user for the number and use it in the PR title and summary.

3. **Validate local state**
   - Run `git status`.
   - If on `main`/`master`, ask for a branch name and create it.
   - If there are local changes, ask: **commit**, **continue without committing**, or **quit**.
   - If commit is chosen, actually run `git add -A`, `git commit -m "<message>"`, then verify with `git status` and `git log -1 --oneline`. If commit fails or is empty, stop.

4. **History safety**
   - Prefer not to rewrite history; rely on GitHub squash merge.
   - Only squash locally if the branch is up to date, commits are low-value/WIP, and all commits are local-only/unshared.
   - If rewriting, create `<branch>-backup` first.
   - Never rewrite shared or behind branches.

5. **Push / detect duplicate PRs**
   - If the branch is ahead of remote, run `git push -u origin <current-branch>`.
   - Never force-push unless history was intentionally rewritten in step 4.
   - If nothing needs pushing, check whether a PR already exists and reuse it instead of creating a duplicate.

6. **Open the PR**
   - Base: default branch. Head: current branch.
   - Title format: `AB#{work_item} | <concise summary>`.
   - Build the body from `.github/pull_request_template.md` and fill only the Quality Checks content from the actual changes.
   - Return the PR URL.

7. **Dispatch deploy + smoke test**
   - Run `deploy-smoke-test.yml` from the default branch.
   - Pass `branch_name=<current-branch>`.
   - Return the run URL.

8. **Report**
   - Return branch, `AB#`, PR URL, and workflow run URL.

## Guardrails

- Run steps in order and stop on the first failure.
- Never update the branch automatically; if behind, stop and ask the user to sync it themselves.
- Prefer GitHub squash merge over local history rewriting.
- Never force-push unless history was intentionally rewritten.
- Always include `AB#` in the PR title.
- Never auto-commit silently; if commit is chosen, actually commit and verify it.
- Never create a duplicate PR.

