# Update Repo

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/jamesbuckett/update-repo?style=social)](https://github.com/jamesbuckett/update-repo/stargazers)
[![Last commit](https://img.shields.io/github/last-commit/jamesbuckett/update-repo)](https://github.com/jamesbuckett/update-repo/commits)
[![Open issues](https://img.shields.io/github/issues/jamesbuckett/update-repo)](https://github.com/jamesbuckett/update-repo/issues)

> A Claude Code skill that scaffolds a GitHub repo with a tailored README, MIT LICENSE, topics, and badges in one shot.

## About

Generates publishable scaffolding for any GitHub-backed git repo in a single pass. Detects the project's intent from manifests, code, and existing metadata, then writes a tailored README with status badges, an MIT LICENSE, a curated set of repository topics, and a clean first commit. Built as a Claude Code skill — install once at `~/.claude/skills/update-repo/` and invoke from any repo by saying "update this repo".

## Quick Start

```bash
git clone https://github.com/jamesbuckett/update-repo.git ~/.claude/skills/update-repo
# Then, inside any GitHub-backed repo, ask Claude to invoke the skill by its trigger phrase.
```

## Usage

Inside any GitHub-backed git repo, ask Claude any of:

- `update this repo`
- `scaffold this repo`
- `set up README and license`
- `make this repo presentable`
- `polish my GitHub repo`
- `set GitHub topics`

The skill detects the repo's intent, surfaces it for confirmation, then writes `README.md`, writes `LICENSE` (MIT), sets repository topics via `gh`, commits with `docs: scaffold README and MIT LICENSE via update-repo`, and pushes.

## Contributing

Issues and pull requests welcome. Please open an issue first to discuss substantial changes.

## License

[MIT](LICENSE) © 2026 James Buckett
