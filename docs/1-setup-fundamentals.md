# Setup fundamentals

## Table of Contents

<!-- toc -->

- [1. What is Cursor?](#1-what-is-cursor)
- [2. Install Cursor](#2-install-cursor)
- [3. Cursor setup](#3-cursor-setup)
- [4. Terminal basics](#4-terminal-basics)
- [5. Package managers](#5-package-managers)
- [6. Git basics](#6-git-basics)
- [7. Troubleshooting](#7-troubleshooting)
- [8. Reference](#8-reference)

<!-- tocstop -->

---

## 1. What is Cursor?

Cursor is an AI code editor. It uses large language models to help you generate, edit, and understand code.

**Key features:**

| Feature          | Description                                           | Shortcut           |
| ---------------  | ----------------------------------------------------- | ------------------ |
| **Tab**          | Multi-line autocomplete that predicts your next edit  | `Tab`              |
| **Inline Edit**  | Select code and describe changes                      | `Cmd+K` / `Ctrl+K` |
| **Agent**        | Build features and make changes across multiple files | `Cmd+I` / `Ctrl+I` |
| **Plan**         | Create step-by-step plans before implementing         | `Cmd+P` / `Ctrl+P` |
| **Ask**          | Ask questions about your codebase                     | User assignable    |
| **Debug**        | Diagnose and fix bugs using runtime traces            | User assignable    |
| **Agent Review** | Review all changes against main branch for issues     | User assignable    |

**Learn more:** [Cursor Concepts][cursor-concepts]

---

## 2. Install Cursor

1. Download [Cursor][cursor-download] and run the installer
2. Create a [GitHub account][github-signup] if you don't have one
3. Enable two-factor authentication (2FA) in your GitHub account settings

> [!NOTE]
> 2FA is required for contributors and recommended for everyone.

---

## 3. Cursor setup

### Getting started

#### First launch

When you open Cursor for the first time:

1. Create a new folder for your project (or open an existing one)
2. Press `Cmd+I` (macOS) or `Ctrl+I` (Windows) to open Agent mode
3. Try a simple prompt like "Create a hello world HTML file"

You're now using Cursor. The sections below explain additional setup and tools as you need them.

#### GitHub integration

Connecting your GitHub account lets you push code to repositories and collaborate with others. Sign in once and Cursor will remember your credentials.

1. Press `Cmd+Shift+P` (macOS) or `Ctrl+Shift+P` (Windows)
2. Type "GitHub: Sign in" and press Enter
3. Authorize Cursor in the browser popup

Once signed in, you can ask the Agent to commit and push changes for you. It writes descriptive commit messages automatically. You can also use the **Source Control** tab in the left sidebar for manual control.

**Docs:** [Cursor GitHub Integration][cursor-github]

### Configuration (optional)

These settings improve your workflow but aren't required to get started.

#### Enable autosave

Add this to your User Settings JSON (`Cmd+Shift+P` > "Preferences: Open User Settings (JSON)"):

```json
"files.autoSave": "afterDelay",
"files.autoSaveDelay": 1000
```

This removes the need to manually save. Changes appear in `git status` automatically.

#### Install CLI command

Open any folder from the terminal with the `cursor` command:

1. Open command palette: `Cmd+Shift+P` (macOS) or `Ctrl+Shift+P` (Windows)
2. Search: "Shell Command: Install 'cursor' command in PATH"
3. Open any folder from terminal:

   ```bash
   cursor .
   ```

#### Project structure

```text
your-project/
├── .cursor/
│   └── rules/
│       └── project.md    # Your project rules
├── .cursorignore         # Files agent should ignore
└── ...
```

#### Sample .cursorignore

```gitignore
node_modules/
dist/
build/
```

**Docs:** [Cursor Rules][cursor-rules]

### Using the Agent

#### Context symbols

Use `@` symbols in prompts to give the agent more context:

- `@files` - browse files or folders
- `@<filename>` - include specific file/folder
- `@docs` - include external documentation
- `@git` - reference git changes and diffs
- `@browser` - open an embedded browser tab for testing and debugging
- `@terminals` - browse terminal windows

> [!TIP]
> You can also attach images by clicking the image button or pasting directly into chat.

#### Auto-run modes

Go to **Settings > Features > Agent** and configure:

| Setting           | Options                                               | Recommended        |
| ----------------- | ----------------------------------------------------- | ------------------ |
| Command execution | Ask Every Time / Auto-Run in Sandbox / Run Everything | **Ask Every Time** |

- **Ask Every Time** - You approve each command before it runs (safest)
- **Auto-Run in Sandbox** - Commands run automatically in a sandbox if possible with restricted network and filesystem access. Otherwise fallback to allowlist.
- **Run Everything (Unsandboxed)** - All commands run automatically without prompts

> [!NOTE]
> The sandbox allows reads and writes within your workspace but blocks network access and git operations by default. You'll still be asked to approve operations that need additional permissions.

#### Keyboard shortcuts

| Action                   | macOS   | Windows  |
| ------------------------ | ------- | -------- |
| Toggle left sidebar      | `Cmd+B` | `Ctrl+B` |
| Open Agent mode          | `Cmd+I` | `Ctrl+I` |
| Open Plan mode           | `Cmd+P` | `Ctrl+P` |
| Switch modes             | `Cmd+.` | `Ctrl+.` |
| Toggle Agent/Editor view | `Cmd+E` | `Ctrl+E` |

---

## 4. Terminal basics

The **terminal** is an application for executing text commands instead of using a graphical interface. The **shell** is the program that interprets your commands (bash, zsh, etc.). On macOS, use the built-in **Terminal** app. On Windows, use **Git Bash** (installed with Git).

> [!NOTE]
> You can use Cursor without knowing terminal commands. The Agent can run commands for you, and the Git tab in the left sidebar handles git visually. Learn these commands when you want more control.

**Commands:**

| Command        | Description                     |
| -------------- | ------------------------------- |
| `pwd`          | Print current directory path    |
| `ls`           | List files in current directory |
| `cd folder`    | Change to specified directory   |
| `cd ..`        | Move up one directory level     |
| `cd ~`         | Change to home directory        |
| `mkdir folder` | Create a new directory          |
| `cat file`     | Display file contents           |

**Keyboard shortcuts:**

| Key    | Action                  |
| ------ | ----------------------- |
| Tab    | Cycle through options   |
| Right  | Autocomplete suggestion |
| Up     | Previous command        |
| Ctrl+C | Cancel current command  |

---

## 5. Package managers

A **package manager** is a tool that installs and updates software from the command line. Instead of downloading installers from websites, you run a single command.

> [!NOTE]
> Package managers are optional but helpful. They make it easier to install tools that your projects need. If you're new to command-line tools, you can skip this step and install software manually when needed.

1. Install a package manager:
   - **macOS:** [Homebrew][brew]
   - **Windows:** [Scoop][scoop]
2. Restart your terminal
3. Install git:

| macOS | Windows |
| ----- | ------- |
| Git comes pre-installed. Run `git --version` to verify. | `scoop install git` |

> [!NOTE]
> **macOS:** If git isn't available, running `git --version` will prompt macOS to install Xcode Command Line Tools automatically.
>
> **Windows:** Use **Git Bash** for terminal commands (installed with Git). WSL works too but adds complexity.

---

## 6. Git basics

**Git** is version control software that tracks changes to your code. It allows you to create commits (saved states), revert to previous versions, upload to hosting platforms like GitHub, and merge work from multiple branches.

> [!TIP]
> You can use Cursor's **Git tab** in the left sidebar and **Timeline** panel to manage git visually without learning commands. The information below is for when you want more control.

### Key concepts

| Term                  | Meaning                                                             |
| --------------------- | ------------------------------------------------------------------- |
| **Repository**        | A folder tracked by Git that contains your project and its history  |
| **Working directory** | Current state of files in your repository                           |
| **Unstaged changes**  | Modified files not yet selected for commit                          |
| **Staged changes**    | Changes selected to be included in the next commit                  |
| **Commit**            | A saved snapshot of staged changes                                  |

> [!NOTE]
> Git stores all version history and metadata in a hidden `.git/` folder inside your repository. You never need to edit this folder directly.

Staging allows you to commit specific changes while leaving others uncommitted. Use `git add <file>` to stage individual files, or `git add .` to stage everything.

### First time setup

Run these once to configure your identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### Essential commands

| Command                     | Description                         |
| --------------------------- | ----------------------------------- |
| `git init`                  | Initialize a new repository         |
| `git clone <url>`           | Download an existing repository     |
| `git status`                | Show modified files                 |
| `git add .`                 | Stage all changes for commit        |
| `git commit -m "msg"`       | Save staged changes with a message  |
| `git push`                  | Upload commits to remote repository |
| `git log --oneline`         | List commit history                 |
| `git log --oneline --graph` | List commit history as a tree       |

> [!TIP]
> Ask Cursor to commit after each completed working change. This makes it easier to revert if needed.

### Inspecting agent changes

When the Agent makes changes, use these commands to see what happened:

```bash
git status                    # Show what files changed
git log --oneline -5         # View recent commit history
git show HEAD -- path/to/file # View the latest committed version of a file
```

### Undo commands

**Discard uncommitted changes:**

> [!WARNING]
> These commands permanently delete uncommitted changes. Commit your work first if you want to keep it.

| Command                          | Description                                          |
| -------------------------------- | ---------------------------------------------------- |
| `git restore .`                  | Discard unstaged changes only (keeps staged changes) |
| `git checkout HEAD -- <file>`    | Restore a single file to its last committed state    |
| `git reset --hard`               | Discard all changes (both staged and unstaged)       |

**Undo commits:**

> [!WARNING]
> Recovery via `git reflog` is sometimes possible if the commit hasn't been garbage collected, but this isn't guaranteed. Use with caution.

| Command                   | Description                            |
| ------------------------- | -------------------------------------- |
| `git reset --soft HEAD~1` | Undo last commit, keep changes staged  |
| `git reset --hard HEAD~1` | Undo last commit, discard changes      |

### Optional: Git GUI

A dedicated Git GUI makes it easier to browse many commits at once in a larger view.

| App                      | Install                          | Notes                           |
| ------------------------ | -------------------------------- | ------------------------------- |
| [Sourcetree][sourcetree] | `brew install --cask sourcetree` | Free, polished. macOS + Windows |

**Docs:** [Git][git-docs]

---

## 7. Troubleshooting

| Problem                     | Solution                                                      |
| --------------------------- | ------------------------------------------------------------- |
| Cursor features not working | Try running `git init`, then close and reopen folder          |
| GitHub authentication fails | Re-run "GitHub: Sign in" from command palette (`Cmd+Shift+P`) |
| Agent can't see my files    | Verify `.cursorignore` is not excluding them                  |
| New packages not working    | Run `npm install` manually after adding dependencies          |

---

## 8. Reference

### Tools

- [Homebrew][brew] (macOS package manager)
- [Scoop][scoop] (Windows package manager)
- [Sourcetree][sourcetree] (Git GUI)

### Documentation

- [Cursor][cursor-docs] - Editor documentation
- [Cursor Concepts][cursor-concepts] - Core features and AI foundations
- [Cursor GitHub Integration][cursor-github] - Connect Cursor to GitHub
- [Cursor Rules][cursor-rules] - Project context rules
- [Git][git-docs] - Version control
- [GitHub][github-docs] - Platform documentation

<!-- Link definitions -->
[brew]: https://brew.sh
[cursor-concepts]: https://cursor.com/docs/get-started/concepts
[cursor-docs]: https://cursor.com/docs
[cursor-download]: https://cursor.com
[cursor-github]: https://cursor.com/docs/integrations/github
[cursor-rules]: https://cursor.com/docs/context/rules
[git-docs]: https://git-scm.com/doc
[github-docs]: https://docs.github.com
[github-signup]: https://github.com/signup
[scoop]: https://scoop.sh
[sourcetree]: https://sourcetreeapp.com
