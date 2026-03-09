---
name: optibot
description: Run AI code reviews with Optibot directly from Claude Code. Use this skill whenever the user mentions code review, optibot, reviewing changes, reviewing a branch, reviewing a diff, or wants feedback on their code before committing or opening a PR. Also use when the user needs help with optibot authentication, API keys, or CI/CD integration. Even if the user just says "review my changes" or "get feedback on this code", this skill applies.
allowed-tools: Bash(optibot *), Bash(which optibot), Bash(npm * optibot), Bash(npx optibot *)
---

# Optibot - AI Code Review from the Terminal

Optibot is a CLI tool that sends code changes to an AI reviewer and returns actionable feedback. Your job is to run reviews on the user's behalf, interpret results, and help them act on the feedback.

## Prerequisites

Before running any optibot command, check if it's installed:

```bash
which optibot
```

If not found, install it:

```bash
npm install -g optibot
```

## Authentication

Optibot needs authentication before running reviews. There are two methods:

**Interactive login** (for humans at a terminal):
```bash
optibot login
```
This opens a browser for OAuth. The token is stored in `~/.optibot/config.json` and lasts 90 days.

**API key** (for CI/CD or headless environments):
Set the environment variable:
```bash
export OPTIBOT_API_KEY=optk_...
```

Or create a key through the CLI:
```bash
optibot apikey create "my-key-name"
```

The key is shown once — save it immediately. Keys always start with `optk_`.

### Managing API keys

```bash
optibot apikey list       # See all keys with creation/last-used dates
optibot apikey delete ID  # Revoke a key by its ID
```

## Running Reviews

There are three modes. Pick the right one based on what the user wants reviewed.

### 1. Review uncommitted local changes

Best for: "review my changes", "check my code before I commit"

```bash
optibot review
```

This runs `git diff HEAD` and sends all changed files for review.

### 2. Review a branch against its base

Best for: "review my branch", "review before I open a PR", "compare against main"

```bash
optibot review -b              # Auto-detects base branch (main/master/develop)
optibot review -b main         # Explicit base branch
optibot review -b origin/main  # Compare against remote
```

The auto-detection tries `origin/main`, then `origin/master`, then `origin/develop`.

### 3. Review an arbitrary diff file

Best for: "review this patch file", "review this diff"

```bash
optibot review --diff path/to/changes.patch
```

## Interpreting Results

The review output has two sections:

**Review Summary** — A general overview of the changes, patterns noticed, and overall assessment.

**File Comments** — Specific feedback tied to file paths and line numbers. Each comment references the exact file and line range. Use these to navigate directly to the code that needs attention.

**Usage counter** — Shows how many reviews have been used out of the daily limit (e.g., `Reviews used: 3/20 (17 remaining)`).

## After a Review

Once you receive the review results, help the user by:

1. Summarizing the key findings in plain language
2. Offering to fix specific issues the review identified — navigate to the mentioned files/lines and apply the suggested changes
3. If the review found no issues, confirm the code looks good

## Workflow Integration

The most powerful pattern is reviewing before committing or opening a PR:

1. User writes code
2. Run `optibot review` (or `optibot review -b` for branch reviews)
3. Read the feedback, fix issues
4. Commit and push

## Troubleshooting

| Error | Meaning | Fix |
|-------|---------|-----|
| Authentication failed (401) | Token expired or invalid API key | Run `optibot login` or check `OPTIBOT_API_KEY` |
| Review limit reached (429) | Daily quota exhausted | Wait for reset (shown in error) or contact getoptimal.ai |
| No seat assigned (403) | User not assigned in their org | Ask org admin to assign a seat |
| Plan doesn't include reviews (402) | Free/basic plan | Upgrade at getoptimal.ai |
| No changes to review | Empty diff | Make some changes first, or use `-b` to compare branches |
| Not a git repository | Running outside a repo | `cd` into a git project first |

## CI/CD Setup

If the user wants optibot in their CI pipeline, suggest this pattern:

```yaml
# GitHub Actions example
- name: Run Optibot Review
  env:
    OPTIBOT_API_KEY: ${{ secrets.OPTIBOT_API_KEY }}
  run: npx optibot review -b main
```

Key points:
- Use `npx optibot` so it doesn't need a global install
- Always use `-b` in CI to compare the PR branch against the base
- Store the API key as a repository secret, never in code
