# Understanding Agent Mistakes

## Table of Contents

- [0. Overview](#0-overview)
- [1. Context management issues](#1-context-management-issues)
- [2. File synchronization problems](#2-file-synchronization-problems)
- [3. Code quality issues](#3-code-quality-issues)
- [4. Workflow best practices](#4-workflow-best-practices)
- [5. Troubleshooting](#5-troubleshooting)
- [6. Reference](#6-reference)

---

## 0. Overview

AI coding agents accelerate development but introduce new failure modes. Understanding these patterns transforms you from a passive user into an effective **Agent Orchestrator**—someone who guides, validates, and corrects AI outputs strategically.

```mermaid
flowchart TB
    subgraph FAILURES["Common Failure Types"]
        A[Context Loss] --> D[Infinite Loops]
        B[File Sync Issues] --> D
        C[Hallucination] --> E[Wrong Code]
    end

    subgraph MITIGATIONS["Mitigation Strategies"]
        F[Clear Context] --> I[Reliable Output]
        G[Save Immediately] --> I
        H[Review & Test] --> I
    end

    FAILURES --> MITIGATIONS
```

### Key principle

Treat AI-generated code as **tentative suggestions from a capable but fallible assistant**. The agent excels at speed and pattern recognition but lacks judgment and self-verification. Your role is to provide the oversight it cannot provide itself.

---

## 1. Context management issues

Context management is the most common source of agent failures. The agent operates within a limited context window—when this fills up or becomes cluttered, performance degrades significantly.

### 1.1 Context loss and infinite loops

**Problem:** The agent loses track of its progress within a task, repeatedly searching for the same information or applying the same fix.

**Symptoms:**
- Repetitive output like "Let me search for..." followed by the same search
- Agent re-asks questions you already answered
- Same code changes applied multiple times

**Solutions:**

| Action | How to do it |
| ------ | ------------ |
| Start fresh | Open a new chat session to reset context |
| Break down tasks | Divide complex work into smaller, focused requests |
| Monitor token usage | Watch for signs of context saturation |
| Clear chat history | Use `Settings > Clear All Chats` periodically |

> [!TIP]
> If the agent enters a loop, don't try to fix it within the same session. Start a new chat with a clearer, more focused prompt.

### 1.2 Hallucination and irrelevant suggestions

**Problem:** The agent generates plausible-looking but incorrect code—referencing non-existent functions, using wrong APIs, or ignoring your actual codebase structure.

**Symptoms:**
- Code references functions or variables that don't exist
- Suggestions that ignore runtime errors or logs you provided
- Generic solutions that don't fit your specific framework or architecture

**Solutions:**

- **Provide explicit context:** Use `@file` or `@folder` to point the agent to relevant code
- **Include error messages:** Paste actual error output rather than describing it
- **Specify constraints:** Mention your framework, versions, and architectural patterns

```
# Bad prompt
"Fix the authentication bug"

# Good prompt
@file:src/auth/login.ts
"The login function throws 'TypeError: Cannot read property email of undefined' 
when the user object is null. Add null checking before accessing user.email."
```

### 1.3 Lost in the middle

**Problem:** When context becomes very long, agents tend to focus on information at the beginning and end, "forgetting" details in the middle.

**Solutions:**

- Keep critical information near the start or end of your prompt
- For long tasks, summarize key constraints at the end
- Reference important files directly rather than relying on earlier mentions

---

## 2. File synchronization problems

### 2.1 Unsaved changes causing loops

**Problem:** The agent applies code changes that register in the editor buffer but aren't immediately written to disk. When the agent re-reads the file, it sees the old content and attempts the same edit again.

**Symptoms:**
- Same edit applied repeatedly
- Unsaved file indicator (dot) persists after AI edits
- Agent "fixes" the same issue multiple times

**Solutions:**

| Setting | Recommendation |
| ------- | -------------- |
| Autosave | Enable `files.autoSave: "afterDelay"` with 1000ms delay |
| Manual save | Press `Cmd+S` / `Ctrl+S` after each AI edit |
| Watch indicators | Check for unsaved dots before allowing agent to continue |

**Enable autosave in settings:**

```json
{
  "files.autoSave": "afterDelay",
  "files.autoSaveDelay": 1000
}
```

> [!WARNING]
> If you notice the agent applying the same change repeatedly, **stop immediately**. Save the file manually, then verify the change was applied before continuing.

### 2.2 Version control as safety net

Always commit working code before starting significant AI-assisted changes. This provides instant rollback capability.

**Quick recovery commands:**

| Situation | Command |
| --------- | ------- |
| Discard all uncommitted changes | `git reset --hard` |
| Undo last commit, keep changes | `git reset --soft HEAD~1` |
| Undo last commit, discard changes | `git reset --hard HEAD~1` |
| View recent commits | `git log --oneline -10` |

> [!TIP]
> Commit after each successful AI-assisted change. Small, frequent commits make it easy to identify exactly where things went wrong.

---

## 3. Code quality issues

### 3.1 The "junior developer" pattern

AI agents can behave like inexperienced developers—producing code that works superficially but contains subtle logic errors, missing edge cases, or architectural problems.

**Common issues:**
- Removing essential null checks or error handling
- Breaking existing functionality while adding new features
- Ignoring project conventions and patterns
- Adding redundant or conflicting dependencies

**Solutions:**

- **Request incremental changes:** Ask for small, focused modifications rather than large rewrites
- **Specify what to preserve:** Explicitly mention code that should not be changed
- **Review diffs carefully:** Check every change before accepting, not just the new code

```
# Risky prompt
"Refactor the user service to use async/await"

# Safer prompt
"In @file:src/services/userService.ts, convert the fetchUser function 
(lines 45-60) to use async/await. Keep all existing error handling 
and don't modify other functions."
```

### 3.2 Enforcing validation with tests

LLMs cannot self-verify their logic. Compensate by having the agent write tests alongside implementation.

**Approach:**

1. Ask the agent to write tests first (TDD style)
2. Then implement the feature
3. Run tests to validate

```
"Using TDD, implement a validateEmail function:
1. First write tests for: valid emails, invalid formats, empty input, null input
2. Then implement the function to pass all tests"
```

> [!NOTE]
> Tests written by AI also need review—but they provide an additional layer of validation and documentation for expected behavior.

---

## 4. Workflow best practices

### 4.1 The orchestrator mindset

Shift from "coder who uses AI" to "orchestrator who directs AI":

| Old approach | Orchestrator approach |
| ------------ | -------------------- |
| Accept suggestions blindly | Review every change critically |
| Fix AI mistakes in same session | Rollback and re-prompt with better context |
| One big request | Multiple small, focused requests |
| Hope it works | Verify with tests and manual review |

### 4.2 Effective prompting strategies

**Structure your prompts:**

1. **Context:** What file/function you're working with
2. **Goal:** What you want to achieve
3. **Constraints:** What should not change, frameworks to use, patterns to follow
4. **Success criteria:** How you'll know it's correct

**Example structured prompt:**

```
Context: @file:src/components/UserProfile.tsx

Goal: Add loading state while fetching user data

Constraints:
- Use existing LoadingSpinner component from @file:src/components/ui/LoadingSpinner.tsx
- Keep current error handling
- Follow existing patterns in the codebase

Success criteria:
- Show spinner during API call
- Hide spinner when data loads or error occurs
- No layout shift when transitioning states
```

### 4.3 When to stop and restart

Recognize when a session has gone off track:

- Agent is looping or repeating itself
- Suggestions have become increasingly irrelevant
- You've corrected the same mistake multiple times
- Context feels "polluted" with failed attempts

**Recovery steps:**

1. Stop the current session
2. Commit or stash any good changes
3. Rollback problematic changes: `git restore .`
4. Start a fresh chat with lessons learned incorporated into the prompt

### 4.4 Use Plan Mode for complex tasks

For multi-step or architectural changes, use Plan Mode (`Cmd+P` / `Ctrl+P`) to:

- Force the agent to research before implementing
- Review the approach before code is written
- Catch misunderstandings early

> [!TIP]
> Plan Mode is especially valuable when you're unsure about the best approach. Let the agent propose a plan, refine it together, then execute.

---

## 5. Troubleshooting

| Problem | Likely cause | Solution |
| ------- | ------------ | -------- |
| Agent repeats same action | Context loss or file sync issue | Start new session, save files manually |
| Suggestions ignore my code | Insufficient context provided | Use `@file` to reference specific files |
| Code references non-existent functions | Hallucination | Provide more explicit constraints and examples |
| Agent removes important code | Unclear about what to preserve | Specify what should NOT change |
| Performance is slow | Large context or heavy codebase | Reduce context, use `.cursorignore` |
| Same "fix" applied repeatedly | Unsaved changes | Enable autosave or save manually after each edit |
| Agent gets stuck on first prompt | Loop bug | Restart Cursor, report to community forum |

---

## 6. Reference

### Key settings

| Setting | Value | Purpose |
| ------- | ----- | ------- |
| `files.autoSave` | `"afterDelay"` | Prevent file sync issues |
| `files.autoSaveDelay` | `1000` | Save after 1 second of inactivity |

### Context symbols

| Symbol | Usage |
| ------ | ----- |
| `@file` | Reference specific file |
| `@folder` | Reference directory |
| `@docs` | Include external documentation |
| `@git` | Reference git changes |

### Chat modes

| Mode | Shortcut | Best for |
| ---- | -------- | -------- |
| Agent | `Cmd+I` / `Ctrl+I` | Implementation tasks |
| Plan | `Cmd+P` / `Ctrl+P` | Complex planning and research |
| Ask | — | Questions about codebase |

### Documentation

- [Cursor Documentation][cursor-docs]
- [Cursor Rules][cursor-rules]
- [Agent Security][cursor-security]
- [Cursor Forum][cursor-forum]

### Community resources

- [Cursor Forum - Agent Issues][forum-issues]

<!-- Link definitions -->
[cursor-docs]: https://docs.cursor.com
[cursor-rules]: https://docs.cursor.com/context/rules
[cursor-security]: https://docs.cursor.com/account/agent-security
[cursor-forum]: https://forum.cursor.com
[forum-issues]: https://forum.cursor.com/c/bug-reports

