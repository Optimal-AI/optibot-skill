# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.3.0] - 2026-09-16

### Added

- README coverage of agent review mode: a "Review as an agent" entry in What It Does, a row in the Usage table, and an Agent review mode section giving the command, what it returns, the CLI 0.8.0 minimum, and a pointer to the same review through the Optibot MCP server's `review_agent` tool.
- The skill's `description` frontmatter names agent review mode, so Claude Code loads the skill when a coding agent wants structured findings rather than only when a person asks for a review.

### Changed

- Agent-mode finding ids are described as labels within one response rather than as stable keys. The service derives an id from the reviewer's own wording and the reviewer rephrases itself on every call, so the same defect comes back under a different id on the next round. The resubmit guidance now says to compare the file, the line range, and the category instead.
- `meta` in the documented `--json` response shape names its `model` and `provider` fields, which the service always sends.
- The Agent review mode section states that agent mode needs CLI 0.8.0 or later, and what to tell the user when an older CLI answers `--agent` with `unknown option`. The file already gave a minimum version for `optibot setup ci`.
- `missingContext`, `reviewCount`, `isOptibotInstalled`, and `meta` are marked optional in the documented response shape. The CLI types them as optional because older and self-hosted backends may omit them, so an agent has to check each one is present before reading it. A response with nothing to ask for omits `missingContext` entirely rather than sending an empty array.
- The documented `category` values are the service's real closed set — `bug`, `security`, `performance`, `refactor`, `tech-debt`, `duplicate`, `style`, `documentation`, `test`, `other`. The previous example named `maintainability`, which the service never returns, and the resubmit guidance asks an agent to match findings on category.
- Interpreting Results explains the unlimited-quota sentinel: an account with no daily cap gets `9007199254740991` for `reviewCount.limit` and `reviewCount.remaining`, and the agent should say there is no daily limit instead of reporting that number.

### Fixed

- `.claude-plugin/marketplace.json` had been left at 1.0.0 while `.claude-plugin/plugin.json` moved with each release. Both now carry the same version, so the marketplace listing matches the plugin it installs.

## [1.2.1] - 2026-08-21

### Added

- A "Ways to run agent mode (and when to use each)" overview near the top of the Agent review mode section, orienting a coding agent among its choices: the raw agent review, the missing-context resubmit (noting the CLI also auto-resubmits), pre-attached `--related`/`--diagnostics` context, and full mode for a human reader. Each entry cross-references the existing detailed section rather than duplicating it.
- A note that the same agent-mode review is available through the Optibot MCP server's `review_agent` tool for MCP hosts such as Cursor and Claude Desktop, where the host re-calls the tool with `relatedPaths` when the reviewer reports missing context.

## [1.2.0] - 2026-08-17

### Added

- Agent review mode guidance in SKILL.md: when the caller is a coding agent that already holds the working copy, it runs `optibot review --agent --json` (optionally with `--related` and `--diagnostics`) for fast, structured findings instead of human-facing prose.
- The `missingContext` resubmit loop: read the named files and re-run with `--related`, capped at about two rounds (each round spends one review from the daily quota).

### Changed

- The Interpreting Results section now covers agent mode's structured `AgentReviewResponse` findings alongside the existing full-mode Summary and File Comments guidance.

## [1.1.1] - 2026-05-20

### Changed

- SKILL.md no longer embeds GitHub Actions YAML inline — CI setup guidance now points at the canonical docs and the `optibot setup ci` flow.
- Reworded the older-CLI fallback so it references the per-provider contract instead of the removed inline snippet.

## [1.1.0] - 2026-04-30

### Changed

- Restructured the Authentication section as a decision tree: dev-machine login, CI/CD setup, and "already inside a CI runner" each get explicit guidance.
- The skill now routes CI/CD-related requests (GitHub Actions, GitLab CI, Jenkins, `OPTIBOT_API_KEY`) through `optibot setup ci` — the new guided CI onboarding command in CLI 0.4.0.
- CI/CD section updated to reflect that the canonical setup is `optibot setup ci`. The GitHub Actions template remains as a reference for users who already have a key.
- Added explicit fallback note for users on CLI < 0.4.0 (use `optibot apikey create <name>` instead).

## [1.0.0] - 2026-03-11

### Added

- Initial release of the Optibot Claude Code plugin
- Skill for running AI code reviews (local changes, branch comparison, diff files)
- Authentication support (interactive login and API key)
- API key management (create, list, delete)
- CI/CD integration guidance (GitHub Actions)
- Troubleshooting reference table
