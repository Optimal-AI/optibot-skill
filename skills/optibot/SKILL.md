---
name: optibot
description: Run AI code reviews with Optibot. Use when the user wants to review code changes, compare branches, review diffs, manage authentication or API keys, or set up Optibot in CI/CD (GitHub Actions, GitLab CI, Jenkins). For CI/CD requests, route through `optibot setup ci`.
allowed-tools: Bash(optibot *), Bash(optibot setup ci *), Bash(which optibot), Bash(npm install -g @optimalai/optibot), Bash(npm install @optimalai/optibot), Bash(npx @optimalai/optibot *), Bash(cat ~/.optibot/config.json), Bash(test -f ~/.optibot/config.json *), Bash(echo $OPTIBOT_API_KEY)
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
npm install -g @optimalai/optibot
```

## Authentication

Optibot needs authentication before running reviews. Pick the right path for the situation:

### On a dev machine, just installing

```bash
optibot login
```
Opens a browser for OAuth. New users are guided through account setup (creating an organization) before being redirected back. The token is stored in `~/.optibot/config.json` and lasts 90 days.

### Setting up CI/CD or automation (recommended path for CI questions)

```bash
optibot setup ci
```

This is the **guided flow**. It logs you in if needed, confirms the right organization, mints a long-lived API key, and prints copy-paste snippets for GitHub Actions, GitLab CI, and a generic shell. The key never expires; revoke it later with `optibot apikey delete <id>`.

If `optibot setup ci` fails with `unknown command`, the user is on CLI < 0.4.0. Fall back to:
```bash
optibot apikey create ci-key
```
…and use the GitHub Actions snippet from the [CI/CD Setup](#cicd-setup) section below.

### Already running inside a CI runner

Set `OPTIBOT_API_KEY` from your CI provider's secrets. Run `optibot setup ci` **on a dev machine first** to mint that key — never inside a CI runner. Attempting `optibot login` inside CI will refuse with a clear error in CLI ≥ 0.4.0; on older versions it hangs for 5 minutes waiting for a browser nobody can see.

### Routing trigger phrases

If the user mentions any of the following, run `optibot setup ci` (or fall back to `optibot apikey create <name>` on older CLIs):
- "GitHub Actions" / "GitLab CI" / "Jenkins" / "CircleCI" / "Buildkite"
- "OPTIBOT_API_KEY"
- "set up Optibot in CI" / "automate Optibot" / "CI integration"

### Managing API keys

```bash
optibot apikey list       # See all keys with creation/last-used dates
optibot apikey delete ID  # Revoke a key by its ID
```

## Logout

To remove saved credentials from the local machine:

```bash
optibot logout
```

This deletes the stored token from `~/.optibot/config.json`. After logging out, the user must run `optibot login` or set `OPTIBOT_API_KEY` before running reviews again.

## Checking Auth Status

To check whether the user is currently authenticated, verify the config file:

```bash
test -f ~/.optibot/config.json && cat ~/.optibot/config.json
```

If the file exists and contains a valid `apiKey` or `token` field, the user is authenticated. If the file is missing or empty, they need to log in.

You can also check for the environment variable:

```bash
echo $OPTIBOT_API_KEY
```

If neither the config file nor the environment variable is set, prompt the user to authenticate (see [Authentication](#authentication)).

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

The recommended path is `optibot setup ci` from a dev machine — it logs the user in if needed, mints a long-lived API key bound to the active organization, and prints the `export OPTIBOT_API_KEY=...` line ready to paste into the CI provider's secret store.

The contract for any CI provider is:

```bash
export OPTIBOT_API_KEY=optk_...        # from the secret store
npx -y @optimalai/optibot review -b <base-branch>
```

Key points to surface to the user:
- Always use `npx -y @optimalai/optibot` in CI — no global install needed.
- Always pass `-b <base-branch>` so the CLI knows what to diff against.
- Store the key as a CI secret named exactly `OPTIBOT_API_KEY` — never inline it.
- For GitHub Actions, set `fetch-depth: 0` on the checkout step so the base branch is reachable.
- For GitLab CI, the default checkout doesn't include the target branch — `git fetch origin $CI_MERGE_REQUEST_TARGET_BRANCH_NAME` first.

For provider-specific YAML, point the user at the Optimal AI docs (forthcoming). Don't paste a stale snippet.
