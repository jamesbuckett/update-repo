# Skill Update Repo

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/jamesbuckett/skill-update-repo?style=social)](https://github.com/jamesbuckett/skill-update-repo/stargazers)
[![Last commit](https://img.shields.io/github/last-commit/jamesbuckett/skill-update-repo)](https://github.com/jamesbuckett/skill-update-repo/commits)
[![Open issues](https://img.shields.io/github/issues/jamesbuckett/skill-update-repo)](https://github.com/jamesbuckett/skill-update-repo/issues)

> Claude Code skill that scaffolds a GitHub repo's README, MIT LICENSE, topics, description, and homepage in one shot.

## About

Scaffolds a GitHub-backed git repo's public-facing metadata in a single pass — a tailored `README.md` with status badges, an MIT `LICENSE`, a curated topic list, the repo description on the GitHub card, and the homepage URL when Pages is enabled. Detects the project's intent from manifests, code, and existing GitHub metadata; surfaces it for confirmation; then writes, commits, pushes, and verifies the round-trip on the remote. Installs as a Claude Code skill at `~/.claude/skills/skill-update-repo/` and runs from any repo with the phrase *update this repo*.

## Quick Start

```bash
# Direct install (recommended)
git clone https://github.com/jamesbuckett/skill-update-repo.git ~/.claude/skills/skill-update-repo

# Or: symlink from a working copy (for active development)
git clone https://github.com/jamesbuckett/skill-update-repo.git ~/projects/skill-update-repo
ln -s ~/projects/skill-update-repo ~/.claude/skills/skill-update-repo
```

Then, inside any GitHub-backed repo, ask Claude to invoke the skill by its trigger phrase.

## Usage

From inside any GitHub-backed git repo, any of these phrases trigger the skill:

```text
"update this repo"
"scaffold this repo"
"set up README and license"
"add badges to my README"
```

The skill detects the project's title, one-line description, category, and topics, then confirms the bundle once before writing `README.md` + `LICENSE`, syncing GitHub-side metadata (topics, description, homepage), pushing, and verifying the round-trip on the remote.

## Project Structure

```text
.
├── SKILL.md                  # Skill body Claude Code loads
├── assets/
│   ├── README.template.md    # Template written to each target repo
│   └── LICENSE.template      # MIT license template
├── README.md
└── LICENSE
```

## Contributing

Issues and pull requests welcome. Please open an issue first to discuss substantial changes.

## License

[MIT](LICENSE) © 2026 James Buckett
