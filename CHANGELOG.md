# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.3.0] - 2026-09-16

### Changed

- Agent-mode finding ids are described as labels within one response rather than as stable keys. The service derives an id from the reviewer's own wording and the reviewer rephrases itself on every call, so the same defect comes back under a different id on the next round. The resubmit guidance now says to compare the file, the line range, and the category instead.
- `meta` in the documented `--json` response shape names its `model` and `provider` fields, which the service always sends.

### Fixed

- `.claude-plugin/marketplace.json` had been left at 1.0.0 while `.claude-plugin/plugin.json` moved with each release. Both now carry the same version, so the marketplace listing matches the plugin it installs.

## [1.2.1] - 2026-08-21

### Added

- A "Ways to run agent mode (and when to use each)" overview near the top of the Agent review mode section, orienting a coding agent among its choices: the raw agent review, the agent review plus signal/noise self-report, the missing-context resubmit (noting the CLI also auto-resubmits), pre-attached `--related`/`--diagnostics` context, and full mode for a human reader. Each entry cross-references the existing detailed section rather than duplicating it.
- A note that the same agent-mode review is available through the Optibot MCP server's `review_agent` tool for MCP hosts such as Cursor and Claude Desktop, where the host re-calls the tool with `relatedPaths` when the reviewer reports missing context.

## [1.2.0] - 2026-08-17

### Added

- Agent review mode guidance in SKILL.md: when the caller is a coding agent that already holds the working copy, it runs `optibot review --agent --json` (optionally with `--related` and `--diagnostics`) for fast, structured findings instead of human-facing prose.
- The signal-versus-noise self-report: after an agent-mode review, the agent classifies every finding as a real issue, a valid suggestion, or noise, then prints a block-character signal-versus-noise bar with counts, Signal%/Noise%, and the signal-to-noise ratio against the 5:1 trust threshold, followed by an auditable per-finding table.
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
