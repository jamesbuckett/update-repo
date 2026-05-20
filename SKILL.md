---
name: update-repo
description: Scaffold or refresh a GitHub repository's public-facing metadata in one shot. Use when the user says "update this repo", "scaffold this repo", "set up README and license", "make this repo presentable", "add badges to my README", "update repo topics", "write a README for this project", "add an MIT license", "polish my GitHub repo", or "set GitHub topics". Detects repo intent from manifests and code, surfaces it for confirmation, then writes a fresh README.md (with license/stars/last-commit/issues badges), writes an MIT LICENSE, sets repository topics via gh, and commits and pushes. Runs on $(pwd) — no path argument. Skip only if the user wants a partial change (e.g., "just add one badge", "edit this README section") rather than a full scaffold.
---

# update-repo

This skill takes the current working directory — assumed to be a GitHub-backed git repo — and produces the public-facing scaffolding: a tailored README.md, an MIT LICENSE, the four standard badges, and a curated topic list, then commits and pushes the changes. When it finishes, the repo is ready to share.

The skill is designed to **overwrite**. README.md and LICENSE are regenerated every run, because the user's mental model is "make this repo presentable from whatever state it's in." If the user has hand-edited the README, they should commit that work first or decline at the Phase C confirmation.

The skill is end-to-end on purpose. The user's natural cadence is "I made a thing → make it look like a real repo." Splitting that into "draft README", "write license", "set topics", "commit", "push" multiplies the friction the skill is meant to remove. The Phase C confirmation is the one chokepoint where the user gets to redirect.

---

## Execution flow

Walk through phases A → H in order. Each phase has a clear gate — do not proceed past a gate without satisfying it.

---

### Phase A — Preflight

Run these checks. Abort with a precise message and the user-facing fix command if any fails. Aborts are not failures of the skill; they are the skill correctly refusing to operate in an unsafe state.

```bash
git rev-parse --git-dir >/dev/null 2>&1                # must be inside a git repo
git symbolic-ref -q HEAD >/dev/null 2>&1               # must be on a branch (not detached)
git remote get-url origin                              # must have an 'origin' remote
git remote get-url origin | grep -qE 'github\.com[:/]' # remote must be GitHub
gh auth status                                         # gh must be authenticated
```

Failure messages (use the exact fix command):

| Failure | Message |
|---|---|
| Not a git repo | `update-repo must be run inside a git repo. cwd=$(pwd)` |
| Detached HEAD | `Checkout a branch first: git switch <branch>` |
| No origin | `Add a remote: git remote add origin git@github.com:OWNER/REPO.git` |
| Not GitHub | `update-repo is GitHub-specific. Origin is <url>.` |
| gh not authenticated | `Run: gh auth login` |

Parse `OWNER` and `REPO` from the origin URL. Handle both forms and strip a trailing `.git` if present:

- `git@github.com:owner/repo.git` → `OWNER=owner`, `REPO=repo`
- `https://github.com/owner/repo.git` → `OWNER=owner`, `REPO=repo`

---

### Phase B — Intent detection

Gather signals in roughly cheapest-first order. Stop as soon as enough information has accumulated to fill the intent object; do not exhaustively read every file when the first manifest already tells you what the repo is.

1. `gh repo view --json description,name,url,defaultBranchRef,isPrivate` — one API call, high signal. The GitHub description, if set, is usually the best one-line description.
2. `ls -1` at repo root — the shape of the top level is itself a strong signal.
3. Manifest files. Read whichever exist, in this priority:
   - `package.json` → parse `.name`, `.description`, `.keywords`. Primary language: Node / JS / TypeScript.
   - `pyproject.toml` → Python. Read `[project]` table.
   - `Cargo.toml` → Rust.
   - `go.mod` → Go.
   - `Gemfile` or `*.gemspec` → Ruby.
   - `*.csproj` → .NET / C#.
   - `composer.json` → PHP.
   - `requirements.txt` or `setup.py` → Python (older layout).
   - `Makefile` → generic build target (no language commitment).
4. Claude-skill / plugin signals:
   - `SKILL.md` at root → category `Claude Code skill`.
   - `manifest.json` with a `commands` or `skills` key → category `plugin`.
   - `.claude/` directory present → category `Claude Code configuration`.
5. Single-file site: top-level `index.html` with no manifest → category `educational page` or `web app`.
6. Directory-layout signals: presence of `src/`, `cmd/`, `app/`, `web/`, `docs/`, `examples/`, `tests/`.
7. Primary language by file-extension count:
   ```bash
   find . -maxdepth 3 -type f \
     \( -name '*.ts' -o -name '*.tsx' -o -name '*.py' -o -name '*.go' \
        -o -name '*.rs' -o -name '*.rb' -o -name '*.java' -o -name '*.cs' \
        -o -name '*.sh' -o -name '*.html' -o -name '*.md' \) | head -200
   ```
8. Existing `README.md` first 30 lines (Read). The user has already articulated intent at some point; reuse the wording when it still fits.
9. `git log --oneline -20` — themes from recent commits. Skip if repo has zero commits.

Synthesize an internal intent object with these fields:

| Field | Type | Notes |
|---|---|---|
| `title` | string | Prettified repo name (turn `my-cool-tool` into `My Cool Tool`). |
| `one_line_description` | string ≤ 100 chars | Prefer GitHub description, then manifest description, then synthesized. |
| `category` | enum | `CLI tool`, `library`, `web app`, `educational page`, `demo`, `infrastructure`, `plugin`, `Claude Code skill`, `other`. |
| `primary_language` | enum | `Node`, `Python`, `Go`, `Rust`, `Ruby`, `.NET`, `PHP`, `Shell`, `HTML`, `other`. |
| `proposed_topics` | list of 5–10 | Lowercase, hyphenated. Describe what the repo *is*, not what it *uses*. |

**Do not invent details when signals are absent.** Leave a field empty and ask the user for it in Phase C.

**Empty-repo path**: if the repo has zero commits and zero files beyond `.git/`, skip auto-detection entirely. Jump to Phase C and ask the user for every field directly.

---

### Phase C — Confirm with the user

Use `AskUserQuestion`. Build a single confirmation question whose description includes the inferred bundle:

```
Title:        <title>
Description:  <one_line_description>
Category:     <category>
Language:     <primary_language>
Topics:       <topic-1, topic-2, ...>

About to write README.md, LICENSE, set topics [<list>], commit, and push.
```

Options:

- **Looks right, proceed** — go to Phase D.
- **Let me edit** — drop to a freeform exchange. Capture overrides for any subset of fields, then re-show the confirmation once. Do not loop more than twice; if the user keeps editing, ask them to articulate the intent in their own words and use that verbatim.
- **Cancel** — stop cleanly. Do not write anything.

The confirmation doubles as a dry-run preview. The "About to write…" line is mandatory — it's how the user knows exactly what state will exist after the run.

**Topic validation** (apply before showing):
- Lowercase letters, digits, hyphens only.
- No underscores. Replace with hyphens.
- ≤ 50 chars per topic.
- ≤ 20 topics total.
- Cannot start or end with a hyphen.

Quietly fix violations (lowercase, replace underscores) before surfacing. If a topic still violates after fixup, drop it and pick another from the signal set.

---

### Phase D — Resolve author

Try in order; use the first non-empty value. The GitHub `.name` field is the user's chosen display name (e.g., `James Buckett`), which reads better in a copyright line than a login or a locally-set short username — so prefer it when set, then fall back to local git config, then to login.

```bash
gh api user --jq .name 2>/dev/null    # GitHub profile display name (preferred)
git config user.name                  # local or global git config (fallback)
gh api user --jq .login 2>/dev/null   # GitHub login (last resort)
```

Treat the literal string `null` (what `jq` returns when the field is JSON null) as empty.

If all three return empty, abort with: `Could not resolve an author name for the LICENSE. Run: git config --global user.name "Your Name"`.

Year: current calendar year (`date +%Y`).

---

### Phase E — Write files

Read the templates and substitute placeholders, then write to repo root with the Write tool.

**Templates:**
- `assets/README.template.md` → write to `<repo>/README.md`
- `assets/LICENSE.template` → write to `<repo>/LICENSE`

**README placeholder substitution:**

| Placeholder | Value |
|---|---|
| `{{TITLE}}` | Title from intent. |
| `{{OWNER}}` | Parsed in Phase A. |
| `{{REPO}}` | Parsed in Phase A. |
| `{{ONE_LINE_DESCRIPTION}}` | One-line description from intent. |
| `{{ABOUT_PARAGRAPH}}` | 2–4 sentences derived from intent: what it does, who it's for, the shape (library / CLI / page). |
| `{{QUICK_START_BLOCK}}` | Language-specific block (see below) including the `## Quick Start` heading, or empty string. |
| `{{USAGE_BLOCK}}` | 1–2 short example snippets, or `_Coming soon._` for empty/unknown repos. |
| `{{PROJECT_STRUCTURE_BLOCK}}` | `## Project Structure` heading + a tree of top-level dirs, or empty string when the layout is trivial. |
| `{{YEAR}}` | Current year. |
| `{{AUTHOR}}` | Resolved in Phase D. |

After substitution, collapse any runs of 3+ consecutive newlines down to 2 — empty conditional blocks otherwise leave double-blank lines that look untidy.

**Quick Start blocks by category or primary language.** Each block consists of `## Quick Start`, a blank line, a fenced bash code block with the canonical 2-step bootstrap. Compose from this table; do not invent commands the language doesn't use. **Category takes precedence over language** — a `Claude Code skill` or `plugin` repo gets the install-into-`~/.claude/...` bootstrap regardless of what `.sh` or `.md` files it contains, because the user's mental model for a skill is "install it", not "build it".

| Category / Language | Lines inside the bash fence |
|---|---|
| Claude Code skill | `git clone https://github.com/{{OWNER}}/{{REPO}}.git ~/.claude/skills/{{REPO}}` then `# Then, inside any GitHub-backed repo, ask Claude to invoke the skill by its trigger phrase.` |
| plugin | `git clone https://github.com/{{OWNER}}/{{REPO}}.git ~/.claude/plugins/{{REPO}}` then `# Then restart Claude Code so the plugin is loaded.` |
| Node | `npm install` then `npm start` |
| Python | `pip install -r requirements.txt   # or: pip install -e .` then `python -m <module>` |
| Go | `go build ./...` then `go run ./cmd/<binary>` |
| Rust | `cargo build` then `cargo run` |
| Ruby | `bundle install` then `bundle exec <command>` |
| .NET | `dotnet restore` then `dotnet run` |
| PHP | `composer install` then `php -S localhost:8000 -t public` |
| Shell | `chmod +x <script>.sh` then `./<script>.sh` |
| HTML / single-file / unknown | (empty string — omit the section entirely) |

**LICENSE placeholder substitution:**

| Placeholder | Value |
|---|---|
| `{{YEAR}}` | Current year. |
| `{{AUTHOR}}` | Resolved in Phase D. |

Write LICENSE to `LICENSE` (no extension — that's the GitHub convention).

---

### Phase F — Update repository topics

PUT replaces the full topic list, which gives "set, not append" semantics — that's what the user expects from "update topics".

```bash
gh api -X PUT "repos/${OWNER}/${REPO}/topics" \
  -H "Accept: application/vnd.github+json" \
  -f 'names[]=topic-one' \
  -f 'names[]=topic-two'
# ...one -f flag per topic
```

If the PUT fails (network, 403, rate limit), **do not abort**. Continue with the commit and push, then at the end of Phase H print a warning:

```
Topics were not updated. Retry: gh api -X PUT "repos/${OWNER}/${REPO}/topics" -f 'names[]=…' -f 'names[]=…'
```

The README + LICENSE work has standalone value; losing topics shouldn't unwind it.

---

### Phase G — Commit and push

First, check for unrelated dirty files. Staging unrelated work without asking is exactly the kind of surprise this skill must avoid:

```bash
DIRTY=$(git status --porcelain | grep -v -E '^(M| M|\?\?) (README\.md|LICENSE)$' || true)
```

If `$DIRTY` is non-empty, use `AskUserQuestion` with the question `Other files have uncommitted changes. Stage only README.md + LICENSE, or stage everything?` and options `Stage only README + LICENSE` (default) / `Stage everything` / `Cancel`.

Stage and commit:

```bash
git add README.md LICENSE          # or 'git add -A' if user chose "stage everything"
git commit -m "docs: scaffold README and MIT LICENSE via update-repo"
```

If `git commit` fails because there is nothing to commit (idempotent re-run on a repo that already has the same content), treat as success and skip the push step. Print: `No changes to commit — README and LICENSE already up to date.`

Push:

```bash
BRANCH=$(git symbolic-ref --short HEAD)
if git rev-parse --abbrev-ref "${BRANCH}@{upstream}" >/dev/null 2>&1; then
  git push
else
  git push -u origin "$BRANCH"
fi
```

If push is rejected (non-fast-forward), **do not force-push**. Force-pushing is destructive and unrelated to the skill's purpose. Stop and print:

```
Push was rejected (remote has commits this branch doesn't). Run: git pull --rebase, then re-invoke update-repo.
```

---

### Phase H — Report

Print a brief summary so the user sees the result without scrolling. Keep it under 8 lines.

```
update-repo complete.
  Files written:   README.md, LICENSE
  Topics set:      topic-a, topic-b, topic-c
  Commit:          <short-sha>
  Repo URL:        https://github.com/<owner>/<repo>
```

Append any warnings from earlier phases (e.g., topics PUT failed) on their own line.

---

## Edge cases reference

| Condition | Behavior |
|---|---|
| Not a git repo | Abort in Phase A with `cwd=$(pwd)` and the fix command. |
| Detached HEAD | Abort; user must check out a branch first. |
| No `origin` remote | Abort; user must `git remote add origin …`. |
| Remote is not GitHub | Abort; the skill is GitHub-specific. |
| `gh auth status` fails | Abort; instruct `gh auth login`. |
| Author unresolvable | Abort in Phase D; instruct `git config --global user.name`. |
| Topics PUT fails | Continue; warn at end with retry command. |
| Push rejected | Stop; instruct `git pull --rebase`. |
| Empty repo (no commits, no files) | Skip detection; ask user for all intent fields directly. |
| Zero commits | First push uses `git push -u origin <branch>`. |
| Pre-existing README/LICENSE | Overwrite (per spec); mention in confirmation summary. |
| Idempotent re-run | `git commit` becomes no-op; skip push; topics PUT is idempotent. |
| Unrelated dirty files | Phase G asks whether to stage only README+LICENSE or everything. |

---

## Discipline reminders

- The README is short, scannable, and concrete. No marketing prose. No filler. If a section adds nothing, omit it — that's why Quick Start and Project Structure are conditional.
- Use imperative voice in the About paragraph (`Generates …`, `Provides …`), not third-person (`This tool generates …`). It's more direct and reads better in a hurry.
- One H1 only; everything else is H2.
- Code fences must specify a language (` ```bash`, ` ```js`).
- Topics describe what the repo *is*, not what it *uses*. `cli-tool`, not `argparse`. Five to ten topics is the sweet spot — fewer is fine, more is noise.
- The whole skill should feel like one decisive step. Confirm once in Phase C, then execute without further questions unless something fails or `$DIRTY` is non-empty in Phase G.
- Never use `git push --force`, `--force-with-lease`, or `--no-verify`. The skill is additive scaffolding; if it can't push cleanly, it stops and tells the user how to resolve.
