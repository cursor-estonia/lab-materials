# Lab materials

## Contents

Hands-on guides from setup to advanced workflows. Go in order or jump to what you need.

- [1. Setup fundamentals](docs/1-setup-fundamentals.md)\
  Terminal, package managers, Git basics, and Cursor IDE setup
- [2. Deployment guide](docs/2-deployment-guide.md)\
  Deploy to Vercel, manage environment variables, and compare hosting
- [3. Understanding agent mistakes](docs/3-understanding-agent-mistakes.md)\
  Context limits, infinite loops, and recovery strategies
- [4. Agent security guide](docs/4-agent-security-guide.md)\
  Security vulnerabilities, prompt injection, and hardening practices
- [5. Subagents and hooks](docs/5-subagents-and-hooks.md)\
  Custom subagents, hooks, and scaling patterns

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
