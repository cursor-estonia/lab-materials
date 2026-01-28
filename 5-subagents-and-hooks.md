# Scaling development with hooks and subagents

## Table of Contents

<!-- toc -->

- [1. Overview](#1-overview)
- [2. Subagents](#2-subagents)
- [3. Hooks](#3-hooks)
- [4. Scaling patterns](#4-scaling-patterns)
- [5. Reference](#5-reference)

<!-- tocstop -->

---

## 1. Overview

**Subagents** and **hooks** let you extend Cursor's agent with custom automation and parallel execution.

| Feature | Purpose |
| ------- | ------- |
| **Subagents** | Independent agents that handle discrete parts of a task in parallel with isolated context |
| **Hooks** | Scripts that run before or after agent actions to observe, block, or modify behavior |

> [!NOTE]
> Subagents are new in Cursor 2.4. Hooks have been available since earlier versions.

---

## 2. Subagents

**Subagents** are independent agents that handle discrete parts of a task. They run in parallel, use their own context, and can be configured with custom prompts, tool access, and models.

### Key concepts

| Term                      | Meaning                                                                            |
| ------------------------- | ---------------------------------------------------------------------------------- |
| **Parallel execution**    | Multiple subagents run simultaneously to speed up complex tasks                    |
| **Context isolation**     | Each subagent has its own context window, keeping the main chat clean              |
| **Specialized expertise** | Create reusable, expert subagents for specific domains (e.g., security, testing)   |

> [!NOTE]
> Cursor includes three built-in subagents that run automatically: **Explore** (codebase search), **Bash** (shell commands), and **Browser** (web interaction). You don't need to configure them.

### Custom subagents

Create custom subagents in `.cursor/agents/` (for project-specific) or `~/.cursor/agents/` (for global) directories. Each subagent is a markdown file with YAML frontmatter.

**Configuration:**

| Field           | Required | Description                                                                       |
| --------------- | -------- | --------------------------------------------------------------------------------- |
| `name`          | No       | Unique identifier. Use lowercase letters and hyphens. Defaults to filename.       |
| `description`   | No       | When to use this subagent. Agent reads this to decide delegation.                 |
| `model`         | No       | Model to use: `inherit` or a specific model ID. Defaults to `inherit`.            |
| `readonly`      | No       | If `true`, the subagent runs with restricted write permissions.                   |
| `is_background` | No       | If `true`, the subagent runs in the background without waiting for completion.    |

### Exercise 1: Build a verifier subagent

This subagent acts as a skeptical code reviewer to validate the main agent's work. This pattern is recommended in the [official docs][subagents-docs].

1. **Create the file:** `.cursor/agents/verifier.md`
2. **Add the content:**

    ```yaml
    ---
    name: verifier
    description: Validates completed work. Use after the agent marks a task done to confirm it actually works.
    model: inherit
    ---

    You are a skeptical validator. Your job is to verify that work claimed as complete actually works.

    When invoked:
    1. Identify what was claimed to be completed.
    2. Check that the implementation exists and is functional.
    3. Run relevant tests or verification steps.
    4. Look for edge cases that may have been missed.

    Be thorough and skeptical. Report:
    - What was verified and passed
    - What was claimed but incomplete or broken
    - Specific issues that need to be addressed

    Do not accept claims at face value. Test everything.
    ```

3. **Usage:** After the main agent completes a task, invoke your new subagent:

    ```text
    /verifier confirm the implementation is complete and all tests pass.
    ```

### Exercise 2: Build a test runner subagent

This subagent runs tests and fixes failures.

1. **Create the file:** `.cursor/agents/test-runner.md`
2. **Add the content:**

    ```yaml
    ---
    name: test-runner
    description: Runs tests and fixes failures. Delegate when code changes need validation.
    model: inherit
    ---

    You are a test automation expert.

    When delegated a task:
    1. Run appropriate tests for the changed code.
    2. If tests fail, analyze the output and identify the root cause.
    3. Fix the issue while preserving test intent.
    4. Re-run to verify the fix.

    Report: tests passed/failed, summary of failures, and changes made.
    ```

3. **Usage:** Ask the agent to delegate test running:

    ```text
    Run tests for the auth module using the test-runner subagent.
    ```

> [!TIP]
> Write clear descriptions so the agent knows when to delegate. The agent reads descriptions to decide which subagent fits the task.

**Docs:** [Subagents][subagents-docs]

---

## 3. Hooks

**Hooks** let you observe, control, and extend the agent loop with custom scripts. They run before or after defined stages of the agent's process and can observe, block, or modify its behavior.

### Example use cases

| Use case          | Description                                                              |
| ----------------- | ------------------------------------------------------------------------ |
| **Automation**    | Run formatters, linters, or other checks automatically after edits       |
| **Security**      | Scan for secrets or PII, and gate risky operations like database writes  |
| **Observability** | Log agent actions to get insights into your development process          |

### Hook events

Cursor provides a rich set of events to hook into. Here are some of the most common:

| Event                           | When it fires                          |
| ------------------------------- | -------------------------------------- |
| `sessionStart` / `sessionEnd`   | At the start and end of a session      |
| `preToolUse` / `postToolUse`    | Before and after any tool is used      |
| `beforeShellExecution`          | Before a shell command is executed     |
| `afterFileEdit`                 | After the agent edits a file           |
| `stop`                          | When the agent's work is complete      |

### Configuration

Create a `hooks.json` file in `~/.cursor/` (global) or `<project>/.cursor/` (project-specific).

```json
{
  "version": 1,
  "hooks": {
    "afterFileEdit": [{ "command": "./hooks/format.sh" }]
  }
}
```

> [!NOTE]
> Your hook scripts' exit codes matter:
> - `0`: Success, the agent proceeds.
> - `2`: Block the action.
> - Other: The hook failed, but the action proceeds (fail-open).

### Exercise 3: Build an audit hook

This hook will log all shell commands and file edits to an audit log.

1. **Create `hooks.json`:** In your project's `.cursor/` directory, create `hooks.json`:

    ```json
    {
      "version": 1,
      "hooks": {
        "beforeShellExecution": [
          { "command": ".cursor/hooks/audit.sh" }
        ],
        "afterFileEdit": [
          { "command": ".cursor/hooks/audit.sh" }
        ]
      }
    }
    ```

2. **Create the script:** In `.cursor/hooks/`, create `audit.sh`:

    ```bash
    #!/bin/bash
    INPUT=$(cat)
    echo "$(date): $INPUT" >> .cursor/audit.log
    echo '{}'
    exit 0
    ```

3. **Make it executable:** `chmod +x .cursor/hooks/audit.sh`
4. **Usage:** As the agent works, check the `.cursor/audit.log` file to see the logged events.

### Exercise 4: Build a "grind until tests pass" hook

This powerful pattern uses the `stop` hook to make the agent iterate until a goal is met.

> [!NOTE]
> This example uses [Bun](https://bun.sh) for TypeScript execution. Install with `brew install bun` (macOS) or `npm install -g bun`.

1. **Create `hooks.json`:**

    ```json
    {
      "version": 1,
      "hooks": {
        "stop": [{ "command": "bun run .cursor/hooks/grind.ts" }]
      }
    }
    ```

2. **Create the script:** In `.cursor/hooks/`, create `grind.ts`:

    ```typescript
    import { readFileSync, existsSync } from "fs";

    interface StopHookInput {
      conversation_id: string;
      status: "completed" | "aborted" | "error";
      loop_count: number;
    }

    const input: StopHookInput = await Bun.stdin.json();
    const MAX_ITERATIONS = 5;

    if (input.status !== "completed" || input.loop_count >= MAX_ITERATIONS) {
      console.log(JSON.stringify({}));
      process.exit(0);
    }

    const scratchpad = existsSync(".cursor/scratchpad.md")
      ? readFileSync(".cursor/scratchpad.md", "utf-8")
      : "";

    if (scratchpad.includes("DONE")) {
      console.log(JSON.stringify({}));
    } else {
      console.log(JSON.stringify({
        followup_message: `[Iteration ${input.loop_count + 1}/${MAX_ITERATIONS}] Continue working. Update .cursor/scratchpad.md with DONE when complete.`
      }));
    }
    ```

3. **Usage:** Prompt the agent: `"Fix all failing tests and write DONE in .cursor/scratchpad.md when finished."`

### Exercise 5: Block dangerous commands

This hook uses a `matcher` to block risky commands.

1. **Add to `hooks.json`:**

    ```json
    {
      "version": 1,
      "hooks": {
        "beforeShellExecution": [
          { "command": ".cursor/hooks/block-dangerous.sh", "matcher": "rm -rf|drop table|truncate" }
        ]
      }
    }
    ```

2. **Create the script:** In `.cursor/hooks/`, create `block-dangerous.sh`:

    ```bash
    #!/bin/bash
    echo '{"permission": "deny", "user_message": "This command has been blocked for safety."}'
    exit 0
    ```

> [!WARNING]
> The `matcher` field is a regular expression. If the command matches the regex, the hook runs and blocks the command.

**Docs:** [Hooks][hooks-docs]

---

## 4. Scaling patterns

Once you've got hooks and subagents working, combine them for more ambitious workflows.

### Orchestrator pattern

A parent agent plans the work, then delegates to specialized subagents:

```text
> "Refactor the auth module. Use the implementer subagent for code changes and the verifier subagent to confirm each change works."
```

The parent stays focused on coordination while subagents handle the grunt work in isolated contexts.

### Parallel fan-out

Spawn multiple subagents to work on independent chunks simultaneously:

```text
> "Spawn subagents: one for frontend refactor, one for backend tests—in parallel"
```

For true isolation, use git worktrees (select from the agent dropdown). Each subagent works in its own branch, then you merge the results.

### The Ralph loop

An iterative self-improvement pattern where the agent:
1. Works through tasks (checkboxes in a task file)
2. Commits progress after each step
3. Logs errors and derives "guardrails" from failures
4. Rotates context at token thresholds (e.g., 80k) to stay fresh

The key insight: hooks can detect when the agent is stuck (repeated failures, thrashing) and trigger a context rotation or spawn a fresh subagent to take over.

**Example:** Use a `subagentStop` hook to log results and update progress:

```json
{
  "version": 1,
  "hooks": {
    "subagentStop": [
      { "command": ".cursor/hooks/log-subagent.sh" }
    ]
  }
}
```

The script can parse the subagent's output, update a progress file, and even derive new rules from errors.

> [!TIP]
> For long-running autonomous work, persist learnings in files like `guardrails.md` that the agent reads on each iteration. Errors become future instructions.

See [Ralph Wiggum Cursor][ralph-repo] for a full implementation of this pattern.

---

## 5. Reference

### Skills vs. subagents vs. rules

| Feature       | Use when...                                                                                   |
| ------------- | --------------------------------------------------------------------------------------------- |
| **Rules**     | You need always-on, static context for every conversation                                     |
| **Skills**    | You need dynamic, procedural "how-to" instructions loaded on demand                           |
| **Subagents** | You need context isolation, parallel execution, or specialized expertise across many steps    |

### Documentation

- [Cursor 2.4 Changelog][changelog-docs]
- [Hooks Documentation][hooks-docs]
- [Subagents Documentation][subagents-docs]
- [Agent Best Practices][best-practices-docs]
- [Skills Documentation][skills-docs]
- [Agent Skills Standard][agent-skills-standard-docs]
- [Agent Skills Directory][agent-skills-directory-docs]
- [Ralph Wiggum Cursor][ralph-repo] (iterative loop example)

<!-- Link definitions -->
[changelog-docs]: https://cursor.com/changelog/2-4
[hooks-docs]: https://cursor.com/docs/agent/hooks
[subagents-docs]: https://cursor.com/docs/context/subagents
[best-practices-docs]: https://cursor.com/blog/agent-best-practices
[skills-docs]: https://cursor.com/docs/context/skills
[agent-skills-standard-docs]: https://agentskills.io/home
[agent-skills-directory-docs]: https://skills.sh/
[ralph-repo]: https://github.com/agrimsingh/ralph-wiggum-cursor
