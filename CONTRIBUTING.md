# Contributing to Optibot Skill

Thank you for your interest in contributing to the Optibot Claude Code plugin.

## Reporting Issues

Open an issue at [github.com/Optimal-AI/optibot-skill/issues](https://github.com/Optimal-AI/optibot-skill/issues) with:

- A clear description of the problem or suggestion
- Steps to reproduce (for bugs)
- Your Claude Code version (`claude --version`)

## Submitting Changes

1. Fork the repository
2. Create a feature branch (`git checkout -b my-feature`)
3. Make your changes
4. Test the plugin locally: `claude plugin install --path ./`
5. Commit your changes (`git commit -m "Add my feature"`)
6. Push to your fork (`git push origin my-feature`)
7. Open a Pull Request

## Plugin Structure

```
optibot-skill/
├── .claude-plugin/
│   └── plugin.json        # Plugin metadata
├── skills/
│   └── optibot/
│       └── SKILL.md       # Skill definition and instructions
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

## Guidelines

- Keep `SKILL.md` under 500 lines
- Test changes locally before submitting
- Follow the existing writing style in SKILL.md
- Do not commit secrets, API keys, or `.env` files
- Update `CHANGELOG.md` with your changes under an `[Unreleased]` section

## Code of Conduct

Be respectful and constructive. We follow the [Contributor Covenant v2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).

## Questions?

Open a discussion or email hello@getoptimal.ai.
