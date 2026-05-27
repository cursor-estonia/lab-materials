# MCP vs Skills in Cursor CLI

## Table of Contents

<!-- toc -->

- [1. Overview](#1-overview)
- [2. Create a Skill with `/create-skill`](#2-create-a-skill-with-create-skill)
- [3. Configure a Git provider MCP in your project](#3-configure-a-git-provider-mcp-in-your-project)
- [5. Game project workflow tips](#5-game-project-workflow-tips)
- [6. Troubleshooting](#6-troubleshooting)
- [7. Reference](#7-reference)

<!-- tocstop -->

---

## 1. Overview

Cursor agents can be extended in two complementary ways:

| Capability | What it is | What it gives the agent |
| ---------- | ---------- | ------------------------ |
| **Skill** | A markdown playbook (`SKILL.md`) | Workflow, conventions, and step-by-step instructions |
| **MCP** | A Model Context Protocol server | Typed tools and resources (API calls, live data, actions) |

**Skills teach the agent how to work.**  
**MCP gives the agent new powers.**

They work well together: a Skill can define a repeatable workflow, while a Git provider MCP performs the actual issue/MR/pipeline operations.

MCP may however use more tokens than a skill, since a MCP query is usually more verbose. Sometimes you can create a skill for a CLI tool which may be token effective than a MCP server, but may lack some functionality. Really depends on the tool and use case.

If you're interested more about the technical side of MCPs, you can read the official [documentation](https://modelcontextprotocol.io/docs/getting-started/intro). Basically it is like an API for the agent to use.

```mermaid
flowchart LR
    User[You in Cursor CLI]
    Agent[Agent]
    Skill[Skill SKILL.md]
    MCP[Git MCP server]
    GitHost[GitHub or GitLab]

    User --> Agent
    Agent --> Skill
    Agent --> MCP
    MCP --> GitHost
```

This guide focuses on first creating a skill for reviewing code based on some static rules (something that is repetitive and great for an agent to do). Then we will configure a git MCP server for querying more information about project management and merge requests.

---

## 2. Create a Skill with `/create-skill`

### What gets created

Running `/create-skill` scaffolds a skill folder with `SKILL.md` frontmatter (`name`, `description`) and starter instructions.

| Location | Path | Scope |
| -------- | ---- | ----- |
| Project | `.cursor/skills/<skill-name>/SKILL.md` | Shared with the repo |
| Personal | `~/.cursor/skills/<skill-name>/SKILL.md` | Available in all projects |

### Create a code review skill

Let's create an example skill for reviewing code style based on some static rules (taken from TalTech's general clean code rules in Estonian).

1. Open Cursor CLI (or Agent chat) in your project root.
2. Run:

   ```text
   /create-skill
   ```

3. When prompted, describe the skill. Example prompt:

   ```text
   Create a project skill for general code review:
   - flag DRY violations and duplicated logic
   - evaluate naming, function length, and readability
   - check separation of concerns and single responsibility
   - follow clean code standards from:
     https://javadoc.pages.taltech.ee/code_style/clean-code.html
   ```

4. Confirm the scaffolded path, for example:

   ```text
   .cursor/skills/code-review-basics/SKILL.md
   ```

5. Refine `SKILL.md` so it is actionable. Minimal example:

   ```markdown
   ---
   name: code-review-basics
   description: >-
     Performs general code review with focus on DRY and clean code principles.
     Use when reviewing pull requests, refactors, and new feature implementations.
   ---

   # Code Review Basics

   ## Review focus
   1. Find duplicated logic and suggest shared abstractions (DRY)
   2. Flag unclear naming and suggest clearer alternatives
   3. Point out overly long functions and mixed responsibilities
   4. Check that code is easy to read and reason about

   ## Standards
   - Use TalTech clean code guidance as baseline:
     https://javadoc.pages.taltech.ee/code_style/clean-code.html
   - Prefer concrete findings with file-level references
   - Prioritize correctness and maintainability over style-only notes
   ```

6. Validate the skill:

   ```text
   /code-review-basics
   Review the current changes and list DRY and clean-code issues first.
   ```

> [!TIP]
> If a skill does not appear in the `/` menu in CLI, keep it in `.cursor/skills/` and invoke it explicitly (`/<skill-name>`). You can also ask the agent to follow that skill by name in your prompt.

**Docs:** [Skills][skills-docs]

---

## 3. Configure a Git provider MCP in your project

This section combines **general MCP configuration** with concrete setups for **GitLab** (GitLab.com or self-hosted) and **GitHub**. It's recommended to configure it on project basis.

### 3.1 Create the project config file

Create `.cursor/mcp.json` in your project root:

```text
your-project/
├── .cursor/
│   ├── mcp.json
│   └── skills/
│       └── code-review-basics/
│           └── SKILL.md
└── ...
```

### 3.2 MCP config shape (what the fields mean)

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "package-name"],
      "env": {
        "API_KEY": "${env:API_KEY}"
      }
    }
  }
}
```

Common fields:

| Field | Purpose |
| ----- | ------- |
| `command` | Executable to start the MCP server (`npx`, `node`, `python`, `docker`, etc.) |
| `args` | Arguments passed to the command |
| `env` | Environment variables for the server |
| `envFile` | Load secrets from a file (recommended for tokens) |

### 3.3 Config locations

| Scope | File |
| ----- | ---- |
| Project | `.cursor/mcp.json` |
| Global | `~/.cursor/mcp.json` |

Project config overrides global config for the same server name.

### 3.4 Keep secrets out of git (recommended setup)

Do not commit tokens in `mcp.json`. Use `envFile` (or `${env:...}` interpolation):

`.env` example (add `.env` to `.gitignore`):

```bash
GITLAB_URL=https://gitlab.example.com
GITLAB_TOKEN=glpat_...
GITHUB_PERSONAL_ACCESS_TOKEN=ghp_...
```

For GitLab.com, set `GITLAB_URL=https://gitlab.com`.

### 3.5 GitLab MCP (GitLab.com or self-hosted)

This guide uses `mcp-gitlab` (Python-based, run via `uvx`).

Add this to `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "gitlab": {
      "command": "uvx",
      "args": ["mcp-gitlab"],
      "envFile": "${workspaceFolder}/.env"
    }
  }
}
```

If your self-hosted GitLab uses a self-signed certificate in a controlled environment, your MCP server may support a TLS override variable (check server docs, e.g. `GITLAB_SSL_VERIFY=false`).

### 3.6 Verify GitLab MCP is loaded

1. Open **Settings > Tools & MCP**.
2. Confirm your server appears and is connected (healthy/green state).
3. In Agent chat, ask:

   ```text
   Using GitLab MCP tools, list open merge requests in our main project and summarize pipeline status.
   ```

**Docs:** [MCP][mcp-docs]

---

### 3.7 GitHub MCP

GitHub MCP configuration is the same pattern: pick a server, add it to `.cursor/mcp.json`, keep tokens in `.env`, and test.

Add this server to `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "envFile": "${workspaceFolder}/.env"
    }
  }
}
```

> [!NOTE]
> GitHub also maintains `github/github-mcp-server`. Community packages vary by feature set; start with the example above, then switch if your team standardizes on the official server.

### 3.8 Verify GitHub MCP is loaded

```text
Using the GitHub MCP tools, list open pull requests for this repository and summarize their CI status.
```

### Combine Skill + MCP in one prompt

After Exercise 1 and provider setup:

```text
Follow /code-review-basics and use GitLab MCP to fetch the merge request diff and summarize the top DRY and clean-code risks.
Return findings by severity.
```

This is the intended pattern: Skill defines process, MCP executes provider actions.

### Safety checklist for production repos

- Start with read-only operations (list/search/get) before create/update actions.
- Use narrow token scopes and short-lived tokens where possible.
- Keep tokens in `.env` or secret manager, never in committed config.
- Review MCP tool calls like shell commands before approval.

---

## 5. Workflow tips

Use your new Skill + Git MCP for repeatable studio workflows:

| Workflow | Skill responsibility | MCP responsibility |
| -------- | -------------------- | ------------------ |
| Feature branch | Naming, MR template, review rules | Create/list MRs, fetch diffs, comment |
| Build validation | Define required jobs per platform | Read pipeline status and failed jobs |
| Release candidate | Changelog + approval checklist | List tags/releases, verify green pipelines |
| Hotfix | Fast-track process and owners | Create hotfix branch/MR, monitor CI |

Suggested daily MCP tool usage:

1. List open MRs/PRs for the game repo
2. Check latest pipeline status for active branches
3. Fetch issue/MR details linked to current task
4. Post summary comment after agent completes implementation

---

## 6. Troubleshooting

| Problem | What to check |
| ------- | ------------- |
| MCP server not listed | `.cursor/mcp.json` path, JSON validity, restart Tools & MCP |
| Auth failures | Token value, scopes, expired token, wrong host URL |
| Self-hosted GitLab TLS errors | Instance URL, cert trust settings, server-specific TLS env vars |
| Skill not in `/` menu | Skill path is `.cursor/skills/<name>/SKILL.md`; invoke explicitly with `/<name>` |
| Agent ignores skill | Mention skill directly: `Follow /code-review-basics ...` |
| `npx` fails on Windows | Use Cursor docs pattern: `command: "cmd"` with `args: ["/c", "npx", ...]` |

---

## 7. Reference

### Cursor docs

- [Skills][skills-docs]
- [MCP][mcp-docs]
- [Agent security guide](4-agent-security-guide.md)
- [TalTech clean code standard][taltech-clean-code]

### Example MCP servers

- [GitHub MCP server][github-mcp-server]
- [MCP servers catalog][mcp-servers]
- [mcp-gitlab][mcp-gitlab]

<!-- Link definitions -->
[mcp-docs]: https://cursor.com/docs/mcp
[skills-docs]: https://cursor.com/docs/context/skills
[github-mcp-server]: https://github.com/github/github-mcp-server
[mcp-servers]: https://github.com/modelcontextprotocol/servers
[mcp-gitlab]: https://github.com/vish288/mcp-gitlab
[taltech-clean-code]: https://javadoc.pages.taltech.ee/code_style/clean-code.html
