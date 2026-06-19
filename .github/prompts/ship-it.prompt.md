---
mode: agent
description: Push the current branch, open a PR, run the CI workflow, then update the Azure Boards task and move it to Done.
---

# Ship It 🚀

Run the full release flow for my current changes. Ask me for any value below that
you cannot determine automatically, then execute every step in order and stop if a
step fails.

## Configuration (edit these once)

- **GitHub repo**: infer from `git remote get-url origin` (owner/name).
- **CI workflow file**: `ci.yml`  <!-- change to your real workflow filename -->
- **Azure DevOps organization**: `REPLACE_WITH_YOUR_ADO_ORG`
- **Azure DevOps project**: `REPLACE_WITH_YOUR_ADO_PROJECT`
- **Target "done" state**: `Done`  <!-- e.g. Done / Closed / Resolved, per your process -->
- **Work item ID**: ask me if I didn't provide one.

## Steps

1. **Validate local state**
   - Run `git status`. If there are uncommitted changes, list them and ask whether
     to commit (with a message I provide) or abort.
   - Capture the current branch name from `git rev-parse --abbrev-ref HEAD`.
   - Refuse to continue if the branch is `main` or `master`.

2. **Push the branch**
   - Run `git push -u origin <current-branch>`.

3. **Open the pull request** (GitHub MCP server)
   - Base branch: `main` (confirm if the default branch is different).
   - **Title**: a concise summary of the changes (derive from commits; confirm with me).
   - **Description**: bullet summary of the changes + the line `AB#<work-item-id>`
     so the PR auto-links to the Azure Boards work item.
   - Return the PR URL.

4. **Run the CI workflow** (GitHub MCP server → Actions)
   - Trigger `workflow_dispatch` on the configured workflow file for this branch.
   - Return the workflow run URL.

5. **Update the Azure Boards task** (Azure DevOps MCP server)
   - Add a comment/description linking the PR URL and noting the CI run.
   - Set `System.State` to the configured "done" state.
   - Return the work item URL.

6. **Report back** a short summary with three links: PR, workflow run, work item.

