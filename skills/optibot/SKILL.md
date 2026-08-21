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

Run it with:

```bash
optibot review --agent --json
```

The `--json` flag makes the CLI return machine-readable findings instead of formatted prose. You can optionally add:

- `--related <path>` (repeatable) — extra context files the reviewer should read that are not part of the diff, for example an interface the changed code implements or a caller of the changed function.
- `--diagnostics <file>` — a file containing local `tsc`/`eslint` output, so the reviewer can weigh its findings against what your own tools already report.

**How it differs from full mode.** The plain `optibot review` (and its `-b` / `--diff` variants above) is a thorough, multi-pass review that returns a human-facing Summary plus File Comments. Agent mode is a single fast pass tuned for a coding agent that already has the code: it returns structured findings, trades some recall for higher precision, and is meant to be consumed and acted on programmatically. Reach for full mode when a person is reading the output; reach for agent mode when you are.

### Ways to run agent mode (and when to use each)

There are a few distinct ways to drive a review, and they compose rather than compete. A typical agent-driven review is the raw agent review, followed by the self-report, and a resubmit if the reviewer asked for more context. Use this list to orient yourself, then follow the detailed section each one points to.

- **Raw agent review** (`optibot review --agent --json`) — the fast default. Use it whenever you are a coding agent mid-change and want quick, structured, machine-actionable findings. This is the starting point for every other pattern below.
- **Agent review plus the signal/noise self-report** — the recommended default behavior. After every raw agent review, classify each finding and print the signal-versus-noise report so the user knows how far to trust the results. Always do this; see [Signal vs noise self-report](#signal-vs-noise-self-report-required-after-every-agent-mode-review) for the exact procedure and output format.
- **Missing-context resubmit** — when the response comes back with a non-empty `missingContext` array, the reviewer is telling you it could not see files it needs. Read those files and re-run with `--related`, capped at about two rounds; see [Resubmit when context is missing](#resubmit-when-context-is-missing). Note that the CLI also auto-resubmits on `missingContext` (up to two rounds) on your behalf, so you may already receive the resubmitted result.
- **Pre-attached context** (`--related <path>` / `--diagnostics <file>`) — when you already know which files matter (the interface the change implements, a caller of the changed function, or your local `tsc`/`eslint` output), attach them up front so the reviewer has them on the first pass. This can avoid a resubmit round entirely; the flags are described in the run instructions above.
- **Full mode** (plain `optibot review`) — when you want the broader, slower, server-side review written for a human to read. It covers large, cross-file changes more thoroughly but returns prose a machine cannot act on directly and costs more server time. Reach for it when a person is reading the output; reach for agent mode when you are. See [Running Reviews](#running-reviews) for its variants.

The same agent-mode review is also available through the Optibot MCP server's `review_agent` tool for MCP hosts such as Cursor and Claude Desktop, where the host re-calls the tool with `relatedPaths` when the reviewer reports missing context.

### The `--json` response shape

`optibot review --agent --json` returns an `AgentReviewResponse` object:

```
{
  status,              // "needs_changes" | "looks_good"
  reviewPass,          // boolean — did the change pass overall
  findings: [ ... ],   // structured findings, see below
  summary,             // one-paragraph overview
  missingContext,      // optional string[] — files the reviewer still wants to see
  reviewCount,         // how many reviews you have used today
  isOptibotInstalled,  // whether the repo has an Optibot config
  meta                 // { mode, durationMs, ... }
}
```

Each entry in `findings` has this shape:

```
{
  id,           // stable identifier, e.g. "AF-1a2b3c4d5e" — use this to refer to a finding
  file,         // path relative to the repo root
  startLine,
  endLine,
  inPatch,      // true if the lines are inside the diff, false if in surrounding context
  severity,     // "blocker" | "warning" | "nit"
  category,     // e.g. "bug", "security", "performance", "maintainability"
  message,      // the finding itself
  suggestedFix, // optional — a concrete fix
  confidence    // 1–10, the reviewer's own confidence
}
```

### Signal vs noise self-report (required after every agent-mode review)

After you receive the findings, you must classify every one of them and print a signal-versus-noise report. You hold the working copy, so you are in the best position to say which findings are real. This report keeps the review honest and tells the user at a glance how far they should trust it.

Classify each finding, by its `id`, into exactly one of three buckets:

- **Signal · real issue** — a genuine bug, security hole, correctness error, performance problem, or real maintainability defect, confirmed by opening the cited file and lines.
- **Signal · valid suggestion** — a technically sound and actionable point, not a false alarm, even if it is not strictly a bug.
- **Noise** — a finding that is wrong, hallucinated, misreads the code, is not actionable, duplicates another finding, or is pure bikeshedding the project does not want.

**Discipline (do not skip this).** You must open the cited file and lines before you label a finding — never classify from the message text alone. When you are genuinely unsure, label it **Noise**. Precision is the axis that matters here: it is better to under-count signal than to inflate it, because a single wrong "signal" label erodes the user's trust in the whole report. This is your own judgment of your own tool's output, so treat the number as a directional dogfooding signal, not a benchmark score.

Then compute the three summary numbers:

- **Signal %** = (real issues + valid suggestions) / total findings.
- **Noise %** = noise / total findings.
- **Signal-to-noise ratio (SNR)** = (real issues + valid suggestions) / noise. The industry trust threshold is about **5:1**; below that, developers start ignoring the tool, so call it out whenever you are under it.

Print the report in exactly this shape — a horizontal signal-versus-noise bar built from block characters, the counts and percentages, the SNR with a warning when it is below 5:1, followed by an auditable per-finding table:

```
📊 Optibot agent review — signal vs noise   (10 findings)

Signal ▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░  70%   7/10
   🐞 real issues        4
   💡 valid suggestions  3
Noise  ▓▓▓▓▓▓░░░░░░░░░░░░░░  30%   3/10

Signal-to-noise ratio: 2.3 : 1   ⚠ below the 5:1 trust threshold
```

| # | id | file:line | severity | agent verdict | why |
|---|----|-----------|----------|---------------|-----|
| 1 | AF-1a2b3c4d5e | `src/auth.ts:42-45` | blocker | 🐞 real issue | Missing null check reachable from the login path. |
| 2 | AF-9f8e7d6c5b | `src/util.ts:10` | nit | ✗ noise | Style preference the repo's prettier config already enforces. |

Rules for rendering the report:

- The bar is 20 characters wide. On the Signal line, fill the signal fraction with `▓` (rounded to the nearest character) and the remainder with `░`; on the Noise line, do the inverse, filling the noise fraction with `▓`.
- Keep the two breakdown lines (`🐞 real issues`, `💡 valid suggestions`) only on the Signal side, indented under it, with their raw counts.
- When the SNR is at or above 5:1, drop the `⚠` note and print `✓ at or above the 5:1 trust threshold` instead.
- In the verdict column use `🐞` for a real issue, `💡` for a valid suggestion, and `✗` for noise, so the table matches the bar.
- The `why` column is one short, complete sentence per finding — the reason for the verdict, grounded in what you actually saw when you opened the lines.

### Resubmit when context is missing

If the response's `missingContext` array is non-empty, the reviewer is telling you it could not see files it needs to judge the change fairly. Read each named file, then re-run agent mode passing those files as context:

```bash
optibot review --agent --json --related path/to/first.ts --related path/to/second.ts
```

Each resubmit round spends one review from your daily quota, so do not loop indefinitely — cap it at about **2 rounds**. Findings carry a stable `id` across rounds, so you can tell which are the same as before and which are new. Once `missingContext` comes back empty (or you have hit the 2-round cap), classify the final set of findings and print the signal-versus-noise report described above.

## Interpreting Results

### Full mode (`optibot review`)

The full-mode review output has two sections:

**Review Summary** — A general overview of the changes, patterns noticed, and overall assessment.

**File Comments** — Specific feedback tied to file paths and line numbers. Each comment references the exact file and line range. Use these to navigate directly to the code that needs attention.

**Usage counter** — Shows how many reviews have been used out of the daily limit (e.g., `Reviews used: 3/20 (17 remaining)`).

### Agent mode (`optibot review --agent --json`)

Agent mode does not return the Summary and File Comments prose. It returns the structured `AgentReviewResponse` described in [Agent review mode](#agent-review-mode): a `findings` array (each finding carries `id`, `file`, `startLine`/`endLine`, `severity`, `category`, `message`, an optional `suggestedFix`, and a `confidence` score), a one-paragraph `summary`, an overall `status` and `reviewPass`, and `reviewCount` for the daily quota. Read the findings directly instead of parsing prose: sort them by `severity` (`blocker`, then `warning`, then `nit`), open each cited `file` at `startLine`-`endLine`, and weigh each finding's `confidence` when deciding what to act on. Always finish an agent-mode review with the signal-versus-noise self-report.

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
