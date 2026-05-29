# Subagents and hooks

## Table of Contents

<!-- toc -->

- [1. Overview](#1-overview)
- [2. Subagents](#2-subagents)
- [3. Hooks](#3-hooks)
- [4. Scaling patterns](#4-scaling-patterns)
- [5. Troubleshooting](#5-troubleshooting)
- [6. Reference](#6-reference)

<!-- tocstop -->

## 1. Overview

**Subagents** and **hooks** extend Cursor's agent: subagents run tasks in parallel, hooks run scripts before or after agent actions.

| Feature       | Purpose                                                                                   |
| ------------- | ----------------------------------------------------------------------------------------- |
| **Subagents** | Independent agents that handle discrete parts of a task in parallel with isolated context |
| **Hooks**     | Scripts that run before or after agent actions to observe, block, or modify behavior      |

> [!NOTE]
> Subagents were introduced in Cursor 2.4. Hooks have been available since earlier versions.

## 2. Subagents

### Key concepts

| Term                      | Meaning                                                                          |
| ------------------------- | -------------------------------------------------------------------------------- |
| **Parallel execution**    | Multiple subagents run simultaneously to speed up complex tasks                  |
| **Context isolation**     | Each subagent has its own context window, keeping the main chat clean            |
| **Specialized expertise** | Create reusable, expert subagents for specific domains (e.g., security, testing) |

> [!NOTE]
> Cursor includes built-in subagents that run automatically: **Explore** (codebase search), **Bash** (shell commands), and **Browser** (web interaction). Check the [docs][subagents-docs] for current capabilities.

### Custom subagents

Create custom subagents in `.cursor/agents/` (for project-specific) or `~/.cursor/agents/` (for global) directories. Each subagent is a markdown file with YAML frontmatter.

**Configuration:**

| Field           | Required | Description                                                                    |
| --------------- | -------- | ------------------------------------------------------------------------------ |
| `name`          | No       | Unique identifier. Use lowercase letters and hyphens. Defaults to filename.    |
| `description`   | No       | When to use this subagent. Agent reads this to decide delegation.              |
| `model`         | No       | Model to use: `inherit` or a specific model ID. Defaults to `inherit`.         |
| `readonly`      | No       | If `true`, the subagent runs with restricted write permissions.                |
| `is_background` | No       | If `true`, the subagent runs in the background without waiting for completion. |

### Exercise 1: Build a verifier subagent

Validates completed work before accepting it as done.

1. **Create the file:** `.cursor/agents/verifier.md`
2. **Add the content:**

   ```yaml
   ---
   name: verifier
   description: Validates completed work. Use after the agent marks a task done.
   model: inherit
   ---

   You are a skeptical validator. When invoked:

   1. Identify what was claimed as completed
   2. Verify the implementation exists and works
   3. Run tests and check edge cases
   4. Report what passed, what failed, and what needs fixing

   Do not accept claims at face value; verify before confirming.
   ```

3. **Usage:** After the main agent completes a task, invoke your new subagent:

   ```text
   /verifier confirm the implementation is complete and all tests pass.
   ```

> [!TIP]
> Write clear descriptions so the agent knows when to delegate.

**Docs:** [Subagents][subagents-docs]

## 3. Hooks

### Example use cases

| Use case          | Description                                                             |
| ----------------- | ----------------------------------------------------------------------- |
| **Automation**    | Run formatters, linters, or other checks automatically after edits      |
| **Security**      | Scan for secrets or PII, and gate risky operations like database writes |
| **Observability** | Log agent actions to get insights into your development process         |

### Hook events

Common events:

| Event                         | When it fires                      |
| ----------------------------- | ---------------------------------- |
| `sessionStart` / `sessionEnd` | At the start and end of a session  |
| `preToolUse` / `postToolUse`  | Before and after any tool is used  |
| `beforeShellExecution`        | Before a shell command is executed |
| `afterFileEdit`               | After the agent edits a file       |
| `stop`                        | When the agent's work is complete  |

### Configuration

Create a `hooks.json` file in `~/.cursor/` (global) or `<project>/.cursor/` (project-specific). Each hook specifies an event and a command to run.

> [!NOTE]
> Hooks communicate via exit codes: `0` = proceed, `2` = block, other = fail-open (proceed anyway).
> To block an action, either exit with code `2` or output `{"permission": "deny"}`.

### Exercise 2: Build an audit hook

Logs all shell commands and file edits.

1. **Create `hooks.json`:** In your project's `.cursor/` directory, create `hooks.json`:

   ```json
   {
     "version": 1,
     "hooks": {
       "beforeShellExecution": [{ "command": ".cursor/hooks/audit.sh" }],
       "afterFileEdit": [{ "command": ".cursor/hooks/audit.sh" }]
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

### Exercise 3: Build an autonomous task loop

Uses subagents to keep context fresh and a hook to restart the agent if tasks remain.

> [!NOTE]
> This example uses [Bun](https://bun.sh) for TypeScript execution. See [installation docs](https://bun.sh/docs/installation).

1. **Create `TASKS.md`** at your project root with checkbox format:

   ```markdown
   # Tasks

   Goal: Add test coverage

   - [ ] Add tests for the main utility functions
   - [ ] Add tests for API endpoint responses
   - [ ] Add tests for error handling cases
   - [ ] Add tests for edge cases (empty input, invalid data)
   - [ ] Ensure all tests pass
   ```

2. **Create the implementer subagent:** In `.cursor/agents/`, create `implementer.md`:

   ```yaml
   ---
   name: implementer
   description: Implements tasks from TASKS.md. Delegate coding work to this agent for fresh context.
   model: inherit
   ---

   You are an implementation specialist. When delegated:

   1. Read TASKS.md to see current progress
   2. Implement the next unchecked task
   3. Mark it complete with [x] in TASKS.md
   4. Run tests and report status
   ```

3. **Set up `hooks.json`:**

   ```json
   {
     "version": 1,
     "hooks": {
       "stop": [
         { "command": "bun run .cursor/hooks/check-tasks.ts", "loop_limit": 10 }
       ]
     }
   }
   ```

   The hook script enforces the iteration cap via `MAX_ITERATIONS`, reading `loop_count` from stdin.

4. **Create the hook script:** In `.cursor/hooks/`, create `check-tasks.ts`:

   ```typescript
   import { readFileSync, existsSync as fileExists } from "fs";

   // When a Cursor agent's turn ends, the IDE runs any registered "stop" hooks.
   // It pipes a JSON payload to the hook's stdin describing what happened,
   // and reads the hook's stdout for instructions on what to do next.
   // Responding with {} means "stop here." Responding with { followup_message }
   // means "start another agent turn with this message."

   interface StopHookInput {
     conversation_id: string;
     status: "completed" | "aborted" | "error";
     loop_count: number; // how many times this hook has already triggered a follow-up (starts at 0)
   }

   // Configuration
   const MAX_ITERATIONS = 10;
   const tasksPath = "TASKS.md";

   // Read the payload Cursor piped to stdin
   const input: StopHookInput = await Bun.stdin.json();

   // An empty JSON response tells Cursor "do not continue"
   const noFollowup = JSON.stringify({});

   const stopLoop = () => {
     console.log(noFollowup);
     process.exit(0);
   };

   // Parse TASKS.md and return a followup message if work remains, or null if done
   const getFollowup = () => {
     const content = readFileSync(tasksPath, "utf-8");
     const completedTasks = (content.match(/- \[x\]/gi) || []).length;
     const remainingTasks = (content.match(/- \[ \]/g) || []).length;
     const totalTasks = completedTasks + remainingTasks;

     if (remainingTasks > 0) {
       return `[${completedTasks}/${totalTasks} done] ${remainingTasks} tasks remain. Delegate to /implementer to continue.`;
     }
     return null;
   };

   // If the agent completed successfully, we haven't looped too many times,
   // and the tasks file exists, check if there's more work to do.
   if (
     input.status === "completed" &&
     input.loop_count < MAX_ITERATIONS &&
     fileExists(tasksPath)
   ) {
     const followup = getFollowup();
     if (followup) {
       console.log(JSON.stringify({ followup_message: followup }));
     } else {
       stopLoop();
     }
   } else {
     stopLoop();
   }
   ```

5. **Usage:** Prompt the agent:

   ```text
   Complete all tasks in TASKS.md. Use /implementer for each batch of work to keep context fresh.
   ```

> [!TIP]
> The IDE has built-in task management, so this file-based approach is more practical for CLI workflows. Test in the IDE to understand the mechanics. See [Ralph Wiggum Cursor][ralph-repo] for a full CLI implementation.

### Exercise 4: Block commands with matcher

Uses a `matcher` to block commands matching a pattern.

1. **Set up `hooks.json`:**

   ```json
   {
     "version": 1,
     "hooks": {
       "beforeShellExecution": [
         {
           "command": ".cursor/hooks/block-command.sh",
           "matcher": "BLOCK_TEST"
         }
       ]
     }
   }
   ```

2. **Create the script:** In `.cursor/hooks/`, create `block-command.sh`:

   ```bash
   #!/bin/bash
   echo '{"permission": "deny", "user_message": "This command has been blocked for safety."}'
   exit 0
   ```

3. **Make it executable:** `chmod +x .cursor/hooks/block-command.sh`

4. **Test it:** Run `echo BLOCK_TEST` and verify it's blocked. The `BLOCK_TEST` pattern is included for safe testing.

> [!WARNING]
> The `matcher` field is a regular expression. If the command matches the regex, the hook runs and blocks the command.

**Docs:** [Hooks][hooks-docs]

## 4. Scaling patterns

Combine hooks and subagents to automate multi-step workflows.

> [!NOTE]
> Cursor now has a built-in **Multitask** mode that coordinates parallel background subagents. It postdates this material and automates some of the manual orchestration shown here.

### Orchestrator pattern

The agent can delegate to defined subagents when prompted. For complex tasks, specify which subagents to use:

```text
> "Refactor the auth module. Use subagents: /implementer for code changes and /verifier to confirm each change works."
```

The parent agent coordinates while subagents do the work in isolated contexts.

> [!NOTE]
> Without git worktrees, parallel agents share the same working directory, so a `git reset` from one can undo another's changes. Worktrees give each agent its own branch and directory (select from the agent dropdown); you merge the results afterward.
> Worktrees do not isolate the environment. For that (separate filesystem, processes, and network), run each agent in its own container or VM (for example, a dev container, or a remote machine over SSH).

## 5. Troubleshooting

| Problem                              | Likely cause                            | Solution                                                                              |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------------------------------- |
| Subagent not available in `/` menu   | Wrong path or filename                  | Place the file in `.cursor/agents/` (project) or `~/.cursor/agents/` (global)         |
| Agent never delegates to a subagent  | Vague `description` frontmatter         | Describe clearly when to use it, or invoke explicitly with `/<name>`                  |
| Hook never runs                      | `hooks.json` not found or invalid JSON  | Confirm it is in `.cursor/`, validate JSON, and match the correct event name          |
| Hook script does not execute         | Script not executable                   | Run `chmod +x .cursor/hooks/<script>` and verify the command path in `hooks.json`     |
| Command not blocked by a hook        | Hook exited with non-zero, non-`2` code | Exit with code `2` or output `{"permission": "deny"}`; check the `matcher` regex      |
| Task loop never stops                | `loop_count` not checked against a cap  | Enforce `MAX_ITERATIONS` in the script and set `loop_limit` on the `stop` hook        |
| Parallel agents overwrite each other | Subagents share one working directory   | Use git worktrees so each agent gets its own branch and directory, then merge results |

## 6. Reference

### Documentation

- [Cursor Changelog][changelog-docs]
- [Cursor 2.4 release notes (subagents introduction)][changelog-2-4-docs]
- [Hooks Documentation][hooks-docs]
- [Subagents Documentation][subagents-docs]
- [Skills Documentation][skills-docs]
- [Agent Best Practices][best-practices-docs]
- [Ralph Wiggum Cursor][ralph-repo]

<!-- Link definitions -->

[changelog-docs]: https://cursor.com/changelog
[changelog-2-4-docs]: https://cursor.com/changelog/2-4
[hooks-docs]: https://cursor.com/docs/hooks
[subagents-docs]: https://cursor.com/docs/subagents
[skills-docs]: https://cursor.com/docs/skills
[best-practices-docs]: https://cursor.com/blog/agent-best-practices
[ralph-repo]: https://github.com/agrimsingh/ralph-wiggum-cursor
