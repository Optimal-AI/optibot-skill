# Optibot — AI Code Review for Claude Code

A [Claude Code](https://claude.ai/claude-code) plugin that brings [Optibot](https://getoptimal.ai) AI-powered code reviews directly into your coding workflow.

Review local changes, compare branches, and get actionable feedback — all without leaving Claude Code.

## Installation

```bash
claude plugin install optibot
```

## Requirements

- [Claude Code](https://claude.ai/claude-code)
- [Optibot CLI](https://www.npmjs.com/package/optibot) (`npm install -g optibot`)
- An [Optibot account](https://agents.getoptimal.ai/signup) (free to start)

## What It Does

Once installed, Claude Code can:

- **Review your code** — just say "review my changes" and Claude runs the review for you
- **Compare branches** — "review my branch against main" triggers a branch diff review
- **Review patch files** — point it at any `.patch` or `.diff` file
- **Manage API keys** — create, list, and delete keys for CI/CD
- **Fix issues** — Claude reads the review feedback and offers to apply fixes directly

## Usage

After installing the plugin, just ask Claude naturally:

| What you say | What happens |
|---|---|
| "review my changes" | Reviews uncommitted local changes |
| "review my branch" | Compares current branch against main |
| "review this diff" | Reviews an arbitrary patch file |
| "set up optibot" | Walks you through auth setup |
| "create an API key for CI" | Creates and displays a new API key |

## Authentication

**Interactive** — run `optibot login` to authenticate via browser (90-day token).

**CI/CD** — set `OPTIBOT_API_KEY=optk_...` as an environment variable.

## CI/CD Integration

```yaml
# GitHub Actions
- name: Optibot Review
  env:
    OPTIBOT_API_KEY: ${{ secrets.OPTIBOT_API_KEY }}
  run: npx optibot review -b main
```

## Links

- [Optibot Website](https://getoptimal.ai)
- [Optibot CLI on npm](https://www.npmjs.com/package/optibot)
- [Sign Up (Free)](https://agents.getoptimal.ai/signup)
- [Claude Code](https://claude.ai/claude-code)
- [Claude Code Plugins](https://docs.anthropic.com/en/docs/claude-code/plugins)

## License

MIT
