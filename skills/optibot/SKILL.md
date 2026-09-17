---
name: optibot
description: Run AI code reviews with Optibot. Use when the user wants to review code changes, compare branches, review diffs, manage authentication or API keys, or set up Optibot in CI/CD (GitHub Actions, GitLab CI, Jenkins). For CI/CD requests, route through `optibot setup ci`. When you are a coding agent that already holds the working copy and wants structured findings to act on rather than prose for a human, use agent review mode (`optibot review --agent --json`).
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
Then wire the printed `optk_...` value into the user's CI provider as `OPTIBOT_API_KEY` and follow the per-provider contract in the [CI/CD Setup](#cicd-setup) section below (npx invocation + base-branch flag + secret name).

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

There are three modes. Pick the right one based on what the user wants reviewed. These three modes all return prose written for a human reader. If the caller is itself a coding agent that already holds the working copy and wants fast, structured findings, use [Agent review mode](#agent-review-mode) instead.

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

## Agent review mode

Agent review mode is for when the caller is itself a coding agent that already holds the working copy — the diff and the source files are already on disk locally, and it wants fast, structured findings it can act on directly rather than prose written for a human. If you are that agent (for example, you just made changes in this repository and want a second opinion before committing), use agent mode instead of the full review described above.

Agent mode needs CLI **0.8.0 or later**. If `optibot review --agent` fails with `unknown option`, the user is on an older CLI: tell them to run `npm install -g @optimalai/optibot` and use full mode in the meantime.

Run it with:

```bash
optibot review --agent --json
```

The `--json` flag makes the CLI return machine-readable findings instead of formatted prose. You can optionally add:

- `--related <path>` (repeatable) — extra context files the reviewer should read that are not part of the diff, for example an interface the changed code implements or a caller of the changed function.
- `--diagnostics <file>` — a file containing local `tsc`/`eslint` output, so the reviewer can weigh its findings against what your own tools already report.
- `--max-agent-rounds <n>` — advanced: caps how many review rounds the CLI runs, including its automatic resubmit when the reviewer reports `missingContext`. Accepts `1`-`3` (default `2`). `1` disables the auto-resubmit (a single fast pass, cheapest and most deterministic — good for CI); `3` allows one extra round for a large cross-file change. Each round is one billed review, which is why the range is capped.

**How it differs from full mode.** The plain `optibot review` (and its `-b` / `--diff` variants above) is a thorough, multi-pass review that returns a human-facing Summary plus File Comments. Agent mode is a single fast pass tuned for a coding agent that already has the code: it returns structured findings, trades some recall for higher precision, and is meant to be consumed and acted on programmatically. Reach for full mode when a person is reading the output; reach for agent mode when you are.

### Ways to run agent mode (and when to use each)

There are a few distinct ways to drive a review, and they compose rather than compete. A typical agent-driven review is the raw agent review, plus a resubmit if the reviewer asked for more context. Use this list to orient yourself, then follow the detailed section each one points to.

- **Raw agent review** (`optibot review --agent --json`) — the fast default. Use it whenever you are a coding agent mid-change and want quick, structured, machine-actionable findings. This is the starting point for every other pattern below.
- **Missing-context resubmit** — when the response comes back with a non-empty `missingContext` array, the reviewer is telling you it could not see files it needs. Read those files and re-run with `--related`, capped at about two rounds; see [Resubmit when context is missing](#resubmit-when-context-is-missing). Note that the CLI also auto-resubmits on `missingContext` (up to two rounds) on your behalf, so you may already receive the resubmitted result.
- **Pre-attached context** (`--related <path>` / `--diagnostics <file>`) — when you already know which files matter (the interface the change implements, a caller of the changed function, or your local `tsc`/`eslint` output), attach them up front so the reviewer has them on the first pass. This can avoid a resubmit round entirely; the flags are described in the run instructions above.
- **Full mode** (plain `optibot review`) — the deepest review Optibot offers, and the right choice whenever a person is going to read the result. It runs multiple server-side passes, follows a change across files, and writes up what it found as a Summary and File Comments a reviewer can take straight into a pull request or a design discussion. Reach for it for large or cross-cutting changes, and for anything you want a human to sign off on; reach for agent mode when you are the one acting on the findings. See [Running Reviews](#running-reviews) for its variants.

The same agent-mode review is also available through the Optibot MCP server's `review_agent` tool for MCP hosts such as Cursor and Claude Desktop, where the host re-calls the tool with `relatedPaths` when the reviewer reports missing context. That tool needs `@optimalai/optibot-mcp` **1.6.0 or later**; on an earlier version the host has no `review_agent` to call, so use the CLI as described above.

### The `--json` response shape

`optibot review --agent --json` returns an `AgentReviewResponse` object:

```
{
  status,              // "needs_changes" | "looks_good"
  reviewPass,          // boolean — did the change pass overall
  findings: [ ... ],   // structured findings, see below
  summary,             // one-paragraph overview
  missingContext,      // optional string[] — files the reviewer still wants to see;
                       // omitted entirely when it needs nothing
  reviewCount,         // optional — { current, limit, remaining } for today
  isOptibotInstalled,  // optional — whether the repo has an Optibot config
  meta                 // optional — { mode, durationMs, model, provider }
}
```

Each entry in `findings` has this shape:

```
{
  id,           // label for this finding in THIS response, e.g. "AF-1a2b3c4d5e" — not stable between runs
  file,         // path relative to the repo root
  startLine,
  endLine,
  inPatch,      // true if the lines are inside the diff, false if in surrounding context
  severity,     // "blocker" | "warning" | "nit"
  category,     // one of: bug, security, performance, refactor, tech-debt,
                //         duplicate, style, documentation, test, other
  message,      // the finding itself
  suggestedFix, // optional — a concrete fix
  confidence    // 1–10, the reviewer's own confidence
}
```

### Resubmit when context is missing

If the response's `missingContext` array is non-empty, the reviewer is telling you it could not see files it needs to judge the change fairly. Read each named file, then re-run agent mode passing those files as context:

```bash
optibot review --agent --json --related path/to/first.ts --related path/to/second.ts
```

Each resubmit round spends one review from your daily quota, so do not loop indefinitely — cap it at about **2 rounds**. Do not match findings across rounds by `id`: the service derives an id from the reviewer's own wording, and the reviewer rephrases itself on every call, so the same defect comes back under a different id. Compare the file, the line range, and the category instead. Once `missingContext` comes back empty or absent (or you have hit the 2-round cap), you are done.

The CLI already enforces this cap for you: its automatic resubmit is bounded by `AGENT_MAX_ROUNDS` (default **2**), and you can tune that bound with `--max-agent-rounds <1-3>` — set `1` to turn the auto-resubmit off entirely (single pass), or `3` to allow one more round. Through the MCP `review_agent` tool (`@optimalai/optibot-mcp` 1.6.0 or later) there is no such counter: the tool is a single-shot primitive and the host drives every resubmit by re-calling it with `relatedPaths`, so the host owns the round count there.

## Interpreting Results

### Full mode (`optibot review`)

The full-mode review output has two sections:

**Review Summary** — A general overview of the changes, patterns noticed, and overall assessment.

**File Comments** — Specific feedback tied to file paths and line numbers. Each comment references the exact file and line range. Use these to navigate directly to the code that needs attention.

**Usage counter** — Shows how many reviews have been used out of the applicable review limit (e.g., `Reviews used: 3/20 (17 remaining)`).

### Agent mode (`optibot review --agent --json`)

Agent mode does not return the Summary and File Comments prose. It returns the structured `AgentReviewResponse` described in [Agent review mode](#agent-review-mode): a `findings` array (each finding carries `id`, `file`, `startLine`/`endLine`, `inPatch`, `severity`, `category`, `message`, an optional `suggestedFix`, and a `confidence` score), a one-paragraph `summary`, an overall `status` and `reviewPass`, and `reviewCount` for the daily quota. The last four of those (`missingContext`, `reviewCount`, `isOptibotInstalled`, `meta`) are optional, because older and self-hosted backends may omit them: check each one is present before reading it. When the account has no daily cap, `reviewCount.limit` and `reviewCount.remaining` come back as the service's unlimited sentinel, `9007199254740991` (`Number.MAX_SAFE_INTEGER`). Tell the user there is no daily limit rather than reporting that number. Read the findings directly instead of parsing prose: sort them by `severity` (`blocker`, then `warning`, then `nit`), open each cited `file` at `startLine`-`endLine`, and weigh each finding's `confidence` when deciding what to act on. `inPatch` tells you whether the lines are inside the diff or in the surrounding context, which is worth saying out loud when you report a finding on code the user did not touch.

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
| Daily review limit reached (429) | Daily quota exhausted | Wait for the reset shown in the error message |
| Trial review limit reached (429) | The trial's total review allowance is used up | Upgrade at agents.getoptimal.ai/dashboard/billing |
| Review limit reached (429) | The organization's review ceiling has been reached | Contact getoptimal.ai/contact to raise it |
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
