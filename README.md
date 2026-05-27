# Lab materials

| Document                                                             | Description                                                               |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| [1. Setup fundamentals](1-setup-fundamentals.md)                     | Terminal, package managers, Git basics, and Cursor IDE setup              |
| [2. Deployment guide](2-deployment-guide.md)                         | Deploy to Vercel, manage environment variables, and compare hosting       |
| [3. Understanding agent mistakes](3-understanding-agent-mistakes.md) | Context management, infinite loops, and workflow best practices           |
| [4. Agent security guide](4-agent-security-guide.md)                 | Security vulnerabilities, prompt injection, and hardening practices       |
| [5. Subagents and hooks](5-subagents-and-hooks.md)                   | Custom subagents, hooks, and scaling patterns for advanced workflows      |

## Contributing

Requires [mise](https://mise.jdx.dev/) for tool management.

```bash
mise install          # install tools from mise.toml
lefthook install      # set up git pre-commit hook
mise tasks            # list available tasks
mise lint             # check markdown lint + TOC
mise lint-fix         # auto-fix lint issues + regenerate TOC
```

[Issues](../../issues) and PRs welcome.
