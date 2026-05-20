# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
