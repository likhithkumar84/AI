# Create Pull Request

Push the current branch, open a pull request tagged with the Azure Work Item
number (`AB#`), then dispatch the On-Demand **Deploy + Smoke Test** workflow.
Ask me for anything you can't determine automatically, run every step in order,
and **stop on the first failure**.

> Run this in **AI Assistant agent mode** so the terminal/git commands below can execute.

## My input (optional)

If I have text selected in the editor, use it as input for this task — for
example the Azure Work Item number (`AB#1234`), a short change summary, or notes:

$SELECTION

If nothing is selected, ignore the block above and ask me for anything you need.

## Context to gather automatically

- **Repo**: `git remote get-url origin` → `owner/name`.
- **Current branch**: `git rev-parse --abbrev-ref HEAD`.
- **Base branch**: the repo's default branch (usually `main`).
- **Azure Work Item**: `AB#{number}` (ask me if not provided).

## Steps

1. **Gather the Azure Work Item number**
   - Use the `AB#{number}` from **My input / selection** above if present.
   - Otherwise, look in the commit messages or branch name; if still not found, ask me.
   - Store it for the PR title and the final report.

2. **Validate local state**
   - Run `git status`.
   - If on `main`/`master`, ask for a new branch name and create it (`git checkout -b <name>`).
   - If there are uncommitted changes, ask for a commit message and commit them.

3. **Squash multiple commits (only when safe)**
   - Fetch the latest base: `git fetch origin <base-branch>`.
   - Count base commits missing from the branch: `git rev-list --count HEAD..origin/<base-branch>`.
   - **If the branch is NOT up to date** (base has commits the branch lacks):
     - **Do NOT squash.** Update the branch instead (`git merge origin/<base-branch>`
       or `git rebase origin/<base-branch>`), then proceed without squashing.
   - **If the branch IS up to date**:
     - If multiple commits exist on the branch vs. base, squash them into one with a
       meaningful message derived from the commit history.
     - Otherwise, proceed without squashing.

4. **Push the branch**
   - `git push -u origin <current-branch>` (or `git push -f origin <current-branch>` if rebased).

5. **Open the pull request**
   - Base: default branch. Head: current branch.
   - **Title**: `AB#{work_item} | <concise summary>` derived from the commits.
   - **Body**: follow the PR body structure below (from `.github/pull_request_template.md`).
   - Create it with the `gh` CLI:
     ```bash
     gh pr create --base <base-branch> --head <current-branch> \
       --title "AB#<work_item> | <summary>" --body-file <body-file>
     ```
   - Return the PR URL.

6. **Dispatch the On-Demand Deploy + Smoke Test workflow**
   - Workflow file: `deploy-smoke-test.yml` (jobs `deploy` → `smoke-test`, `workflow_dispatch`).
   - Dispatch `ref`: the **default branch** (`main`), where the workflow file lives.
   - Input `branch_name`: the **current branch** (both jobs check out this branch).
     ```bash
     gh workflow run deploy-smoke-test.yml --ref main -f branch_name=<current-branch>
     ```
   - Return the dispatched workflow run URL.

7. **Report** a short summary: branch, Azure Work Item (`AB#`), PR URL, and the
   deployment/smoke-test run URL.

## Pull request body structure

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
