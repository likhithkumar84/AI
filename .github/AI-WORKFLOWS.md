# AI Workflows (Agentic Workflows)

A reference for how the AI-customization pieces in this repo compose into an
**AI workflow**, and how that differs from a **GitHub Actions workflow**.

> TL;DR: A GitHub Actions workflow is deterministic YAML run by GitHub's servers.
> An **AI workflow** is a set of natural-language steps an LLM agent reasons over,
> deciding which tools to call to reach a goal. `create-pull-request.prompt.md`
> in this repo is already an AI workflow.

---

## 1. AI workflow vs GitHub Actions workflow

| Aspect       | GitHub Actions workflow (`.github/workflows/*.yml`) | AI workflow (agentic)                                          |
|--------------|-----------------------------------------------------|----------------------------------------------------------------|
| Driven by    | Fixed YAML rules                                    | An LLM agent reasoning over instructions                       |
| Behavior     | Deterministic (same every run)                      | Adaptive — chooses steps, can branch / ask                     |
| Authored in  | YAML                                                | Natural language (Markdown)                                    |
| Runs on      | GitHub runners                                      | Copilot in the editor **or** an agent in CI                    |
| Triggered by | `on:` events (push, dispatch, schedule)             | You invoke it, or an event hands a task to the agent           |
| Examples     | `ci.yml`, `deploy-smoke-test.yml`                   | `prompts/create-pull-request.prompt.md`, `skills/.../SKILL.md` |

---

## 2. Building blocks of an AI workflow

| Layer        | File / location                          | Role                                          |
|--------------|------------------------------------------|-----------------------------------------------|
| Instructions | `.github/copilot-instructions.md`        | Always-on rules & standards (policies)        |
| Prompt       | `.github/prompts/*.prompt.md`            | On-demand task you invoke with `/name`        |
| Skill        | `.github/skills/<name>/SKILL.md`         | Knowledge the agent auto-loads when relevant  |
| Chat mode    | `.github/chatmodes/*.chatmode.md`        | A persona + scoped toolset for a job          |
| Tools / MCP  | MCP servers, git, gh CLI, terminal, APIs | The "hands" the agent uses                    |
| Agent loop   | (runtime behavior)                       | plan → call tool → observe → iterate → report |

An AI workflow = **steps (natural language) + decision points + stop conditions + tools**.

### Same workflow, different trigger

The "AI workflow" is *what* (the steps the agent runs). Instructions, prompts,
skills, and chat modes are just **containers** that package those steps and decide
**when/how** they fire. A container can hold a full multi-step workflow *or* a simple
one-line task — and you can even run a workflow with no container at all, just by
describing the steps in chat.

| Container    | Who triggers it     | When                                        |
|--------------|---------------------|---------------------------------------------|
| Instructions | Nobody — always on  | Applied to every request                    |
| Prompt       | **You**             | When you type `/name`                       |
| Skill        | **The AI**          | When your request matches its `description` |
| Chat mode    | **You** (select it) | For the whole chat session                  |

---

## 3. Two execution surfaces (how to integrate)

1. **Interactive (in the editor)** — you trigger the prompt/skill; the agent runs
   the steps live and asks you at decision points. *(What this repo uses today.)*
2. **Autonomous (in CI)** — the same natural-language workflow runs unattended on
   GitHub, triggered by an event (issue, label, schedule), using tools with scoped
   permissions, and producing commits / PRs / comments by itself. This is where an
   AI workflow runs **as** a GitHub Actions job.

---

## 4. The workflow we already have

`create-pull-request.prompt.md` (and its sibling `SKILL.md`) encode this agent loop:

```
gather facts ─▶ sync gate ─▶ validate local state ─▶ history safety
       ─▶ push / detect duplicate PR ─▶ open PR ─▶ dispatch deploy ─▶ report
```

- **Decision points:** "ask me for the AB# number", "commit / continue / quit".
- **Stop conditions:** "stop on the first failure", "if behind base, stop and ask".
- **Tools used:** git, GitHub PR API, GitHub Actions dispatch.

---

## 5. The integration seam (AI ➜ GitHub Actions)

Step 7 of the AI workflow **dispatches** the deterministic GitHub Actions workflow:

```
You ──▶ AI workflow (prompt / skill)            ──dispatch──▶  GitHub Actions workflow
        git • PR • decisions • guardrails                       deploy ──▶ smoke-test
        (reasoning; runs in the editor)                         (deterministic; runs on GitHub)
```

Pattern: **the AI does the judgment work, then triggers deterministic automation.**
The `branch_name` the AI passes maps to `inputs.branch_name` in
`deploy-smoke-test.yml`.

---

## 6. How workflows get access — credentials & MCP

Any workflow that touches an external system needs real credentials (API key,
OAuth token, PAT). What differs is **where the secret lives** and **who decides
to use it**.

| Aspect                 | n8n / Zapier                 | AI agent (Copilot)                              |
|------------------------|------------------------------|-------------------------------------------------|
| Unit of access         | Node + stored **Credential** | **Tool**, usually via an **MCP server**         |
| Where the secret lives | Credentials store            | MCP server config / host env — never in the LLM |
| Who triggers the call  | You (wired visually)         | The LLM decides at runtime                      |
| Example                | "GitHub node" → PAT          | GitHub MCP server → PAT → GitHub API            |

**MCP (Model Context Protocol)** is the open "USB port" standard for plugging
tools/data into an agent. An **MCP server** wraps a system (GitHub, a DB, Jira),
**holds its credential**, and exposes tools the agent can call. The LLM never sees
the PAT — it just calls a tool, and the MCP server makes the authenticated API call.

> MCP is the standard way to add **external** access, but not the only way — the
> agent also has **built-in tools** (terminal, file edit, git) provided by the
> editor with no MCP server involved.

---

## 7. How to add a new AI workflow

1. **Define the goal + guardrails** (what "done" looks like, what it must never do).
2. **Pick the surface** — interactive (prompt/skill) or autonomous (CI agent).
3. **Write the steps in natural language** — ordered, with decision points.
4. **List the tools** it may use (git, gh, MCP servers, HTTP).
5. **Add stop conditions** (fail fast, ask on ambiguity).
6. **Integrate** — end with a hand-off (e.g., dispatch a GitHub Actions workflow,
   open a PR, post a comment).
7. **Test interactively first**, then optionally promote to autonomous CI.

