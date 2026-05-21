# Skill Update Repo

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/jamesbuckett/skill-update-repo?style=social)](https://github.com/jamesbuckett/skill-update-repo/stargazers)
[![Last commit](https://img.shields.io/github/last-commit/jamesbuckett/skill-update-repo)](https://github.com/jamesbuckett/skill-update-repo/commits)
[![Open issues](https://img.shields.io/github/issues/jamesbuckett/skill-update-repo)](https://github.com/jamesbuckett/skill-update-repo/issues)

> Claude Code skill that scaffolds a GitHub repo with a tailored README, MIT LICENSE, topics, and badges in one shot.

## About

Generates publishable scaffolding for any GitHub-backed git repo in a single pass. Detects the project's intent from manifests, code, and existing metadata; surfaces it for confirmation; then writes a tailored `README.md` with status badges, an MIT `LICENSE`, a curated topic list set via `gh`, and a clean commit — all from one trigger phrase. Built as a Claude Code skill: install once at `~/.claude/skills/skill-update-repo/` and invoke from any repo by saying *"update this repo"*.

## Prerequisites

- **[Claude Code](https://docs.claude.com/en/docs/claude-code)** — the CLI that loads and runs skills.
- **[GitHub CLI (`gh`)](https://cli.github.com/)**, authenticated (`gh auth login`) — the skill uses it to set repository topics and resolve the author name.
- **git** — to clone this repo and to commit/push from the target repo.
- A target repo that is a GitHub-backed git repo with an `origin` remote pointing at `github.com`.

## Installation

The skill installs as a directory under `~/.claude/skills/`. Claude Code auto-discovers any skill placed there at session start.

### Option A — Direct install (recommended)

For most users. Clones a snapshot of the skill straight into the skills directory.

```bash
git clone https://github.com/jamesbuckett/skill-update-repo.git \
  ~/.claude/skills/skill-update-repo
```

Restart Claude Code (or start a new session) so the skill is registered.

### Option B — Symlink from a working copy (for active development)

Use this if you want to edit the skill and have your changes take effect immediately, without re-cloning or `git pull`-ing inside `~/.claude/skills/`.

```bash
# 1. Clone the repo somewhere convenient.
git clone https://github.com/jamesbuckett/skill-update-repo.git \
  ~/projects/skill-update-repo

# 2. Symlink the working copy into the skills directory.
ln -s ~/projects/skill-update-repo ~/.claude/skills/skill-update-repo
```

If `~/.claude/skills/skill-update-repo` already exists (e.g., from a previous direct install), remove it first:

```bash
rm -rf ~/.claude/skills/skill-update-repo
ln -s ~/projects/skill-update-repo ~/.claude/skills/skill-update-repo
```

Then restart Claude Code. Subsequent edits inside `~/projects/skill-update-repo/` are picked up at the next session start.

### Verify the install

Inside Claude Code, in any directory, ask:

```text
> what skills do you have available?
```

You should see `skill-update-repo` listed with its trigger description. If it's missing, confirm `~/.claude/skills/skill-update-repo/SKILL.md` is readable (not nested one level deeper, not broken-symlink) and restart Claude Code.

### Update

```bash
# Direct install
cd ~/.claude/skills/skill-update-repo && git pull

# Symlink install
cd ~/projects/skill-update-repo && git pull
```

### Uninstall

```bash
# Removes either a direct clone or a symlink; the symlink target is untouched.
rm -rf ~/.claude/skills/skill-update-repo
```

## Usage

Inside any GitHub-backed git repo, ask Claude any of:

- `update this repo`
- `scaffold this repo`
- `set up README and license`
- `make this repo presentable`
- `polish my GitHub repo`
- `set GitHub topics`

The skill walks through eight phases:

1. **Preflight** — verifies you're in a git repo on a branch, with a GitHub `origin`, and `gh` authenticated.
2. **Intent detection** — reads manifests, layout, language signals, and existing README to infer title, description, category, language, and topics.
3. **Confirm with the user** — surfaces the inferred bundle and the exact write plan. You can accept, edit, or cancel.
4. **Resolve author** — uses `gh api user .name`, then `git config user.name`, then `gh api user .login`.
5. **Write files** — substitutes placeholders into `assets/README.template.md` and `assets/LICENSE.template`, writes them to repo root.
6. **Update repository topics** — `PUT repos/OWNER/REPO/topics` (replaces, not appends).
7. **Commit and push** — `docs: scaffold README and MIT LICENSE via skill-update-repo`, then `git push` (or `git push -u origin <branch>` on first push).
8. **Report** — prints a short summary of files written, topics set, commit SHA, and repo URL.

The skill is designed to **overwrite** `README.md` and `LICENSE` every run — the mental model is *"make this repo presentable from whatever state it's in"*. If you have hand-edited content you want to keep, commit it first or decline at the Phase C confirmation.

For the full execution flow, edge-case handling, and the rules for composing Quick Start blocks per language and category, read [`SKILL.md`](SKILL.md).

## Project Structure

```
skill-update-repo/
├── SKILL.md                  — the skill definition Claude Code loads
├── assets/
│   ├── README.template.md    — README template written into target repos
│   └── LICENSE.template      — MIT LICENSE template written into target repos
├── LICENSE                   — this repo's own MIT license
└── README.md                 — this file
```

## Contributing

Issues and pull requests welcome. Please open an issue first to discuss substantial changes.

## License

[MIT](LICENSE) © 2026 James Buckett
