---
name: skill-update-repo
description: Scaffold or refresh a GitHub repository's public-facing metadata in one shot. Use when the user says "update this repo", "scaffold this repo", "set up README and license", "make this repo presentable", "add badges to my README", "update repo topics", "write a README for this project", "add an MIT license", "polish my GitHub repo", or "set GitHub topics". Detects repo intent from manifests and code, surfaces it for confirmation, then writes a fresh README.md (with license/stars/last-commit/issues badges), writes an MIT LICENSE, sets repository topics, syncs the repo description (and homepage when Pages is enabled) via gh, then commits, pushes, and verifies the round-trip on the remote. Runs on $(pwd) — no path argument. Skip only if the user wants a partial change (e.g., "just add one badge", "edit this README section") rather than a full scaffold.
---

# skill-update-repo

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
| Not a git repo | `skill-update-repo must be run inside a git repo. cwd=$(pwd)` |
| Detached HEAD | `Checkout a branch first: git switch <branch>` |
| No origin | `Add a remote: git remote add origin git@github.com:OWNER/REPO.git` |
| Not GitHub | `skill-update-repo is GitHub-specific. Origin is <url>.` |
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

**Quick Start blocks by category or primary language.** Each block consists of `## Quick Start`, a blank line, and a fenced bash code block with the canonical bootstrap commands. Compose from this table; do not invent commands the language doesn't use. **Category takes precedence over language** — a `Claude Code skill` or `plugin` repo gets the install-into-`~/.claude/...` bootstrap regardless of what `.sh` or `.md` files it contains, because the user's mental model for a skill is "install it", not "build it".

| Category / Language | Lines inside the bash fence |
|---|---|
| Claude Code skill | See **Skill / plugin Quick Start blocks** below. |
| plugin | See **Skill / plugin Quick Start blocks** below. |
| Node | `npm install` then `npm start` |
| Python | `pip install -r requirements.txt   # or: pip install -e .` then `python -m <module>` |
| Go | `go build ./...` then `go run ./cmd/<binary>` |
| Rust | `cargo build` then `cargo run` |
| Ruby | `bundle install` then `bundle exec <command>` |
| .NET | `dotnet restore` then `dotnet run` |
| PHP | `composer install` then `php -S localhost:8000 -t public` |
| Shell | `chmod +x <script>.sh` then `./<script>.sh` |
| HTML / single-file / unknown | (empty string — omit the section entirely) |

**Skill / plugin Quick Start blocks.** For these two categories, the Quick Start ships two install options in the same fenced block — a direct clone (recommended) and a symlink-from-a-working-copy variant (for users who want their edits to propagate without re-cloning).

For `Claude Code skill`:

````markdown
## Quick Start

```bash
# Direct install (recommended)
git clone https://github.com/{{OWNER}}/{{REPO}}.git ~/.claude/skills/{{REPO}}

# Or: symlink from a working copy (for active development)
git clone https://github.com/{{OWNER}}/{{REPO}}.git ~/projects/{{REPO}}
ln -s ~/projects/{{REPO}} ~/.claude/skills/{{REPO}}
```

Then, inside any GitHub-backed repo, ask Claude to invoke the skill by its trigger phrase.
````

For `plugin`, substitute `~/.claude/plugins/` for `~/.claude/skills/` and replace the trailing line with `Then restart Claude Code so the plugin is loaded.`

**LICENSE placeholder substitution:**

| Placeholder | Value |
|---|---|
| `{{YEAR}}` | Current year. |
| `{{AUTHOR}}` | Resolved in Phase D. |

Write LICENSE to `LICENSE` (no extension — that's the GitHub convention).

---

### Phase F — Update repository metadata

Three GitHub-side fields get synced in this phase: topics, the repo description shown on the card, and the homepage URL (only when Pages is enabled and no homepage is already set). All three are best-effort — failures warn in Phase H but never abort. The README + LICENSE work has standalone value; losing GitHub-side metadata shouldn't unwind it.

**Topics.** PUT replaces the full list, which gives "set, not append" semantics — that's what the user expects from "update topics".

```bash
gh api -X PUT "repos/${OWNER}/${REPO}/topics" \
  -H "Accept: application/vnd.github+json" \
  -f 'names[]=topic-one' \
  -f 'names[]=topic-two'
# ...one -f flag per topic
```

After the PUT, fetch the list back and compare. If the round-trip diverges (e.g., names normalized to empty, server-side filter, silent rejection), capture a warning for Phase H — do not retry inline.

```bash
EXPECTED=$(printf '%s\n' "${TOPICS[@]}" | sort -u | paste -sd, -)
ACTUAL=$(gh api "repos/${OWNER}/${REPO}/topics" --jq '.names | sort | join(",")' 2>/dev/null || true)
[ "$EXPECTED" = "$ACTUAL" ]   # else: WARN_TOPICS
```

**Description.** The README puts the one-line description in a `>` callout, but the repo card on github.com is a separate field. Sync them so the same sentence appears in both places.

```bash
gh repo edit "${OWNER}/${REPO}" --description "${ONE_LINE_DESCRIPTION}"
```

**Homepage.** Skip entirely if the repo already has a non-empty `homepageUrl` — the user picked one already; don't clobber. Otherwise, if GitHub Pages is enabled, set the homepage to the Pages URL.

```bash
EXISTING=$(gh repo view "${OWNER}/${REPO}" --json homepageUrl --jq '.homepageUrl // ""')
if [ -z "$EXISTING" ]; then
  PAGES_URL=$(gh api "repos/${OWNER}/${REPO}/pages" --jq '.html_url' 2>/dev/null) || PAGES_URL=""
  case "$PAGES_URL" in
    https://*) gh repo edit "${OWNER}/${REPO}" --homepage "$PAGES_URL" ;;
  esac
fi
```

**Critical:** on a 404 (Pages not enabled), `gh api` writes the error JSON body to stdout *and* exits non-zero. Use `|| PAGES_URL=""` (not `|| true`) to reset the variable on failure, and gate the edit on `case https://*)` to refuse any captured value that isn't a real URL. A naive `[ -n "$PAGES_URL" ]` will accept JSON garbage and clobber the repo card with `{"message":"Not Found",...}`.

Do not invent homepage URLs from any other signal (e.g., a `docs/` directory, a guessed deploy URL, the repo name). Pages-enabled is the only signal with no false positives.

If any of topics / description / homepage individually fail (non-zero exit, network error, 403), capture the warning text and continue. Warning lines that may surface in Phase H:

```
Topics were not updated. Retry: gh api -X PUT "repos/${OWNER}/${REPO}/topics" -f 'names[]=…'
Topics on remote do not match what was set. Expected: <list>. Actual: <list>.
Description was not updated. Retry: gh repo edit "${OWNER}/${REPO}" --description "…"
Homepage was not updated. Retry: gh repo edit "${OWNER}/${REPO}" --homepage "…"
```

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
git commit -m "docs: scaffold README and MIT LICENSE via skill-update-repo"
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
Push was rejected (remote has commits this branch doesn't). Run: git pull --rebase, then re-invoke skill-update-repo.
```

After a successful push, verify the commit actually reached the remote at the branch tip. This catches edge cases where `git push` succeeded locally but server-side rules (branch protections, pre-receive hooks, squash policies) rewrote or rejected the commit silently.

```bash
LOCAL_SHA=$(git rev-parse HEAD)
REMOTE_SHA=$(gh api "repos/${OWNER}/${REPO}/branches/${BRANCH}" --jq '.commit.sha' 2>/dev/null || true)
[ "$LOCAL_SHA" = "$REMOTE_SHA" ]   # else: WARN_PUSH
```

If the SHAs diverge, capture a warning for Phase H. Do not roll back the local commit — the user can compare the two and decide.

```
Pushed but remote tip does not match local HEAD. Local: <sha>. Remote: <sha>. Inspect with: git log origin/${BRANCH}..HEAD
```

Skip this verification on the idempotent-re-run path (the "nothing to commit" branch printed earlier) — there's nothing to verify.

---

### Phase H — Report

Print a brief summary so the user sees the result without scrolling. Keep it under 10 lines.

```
skill-update-repo complete.
  Files written:    README.md, LICENSE
  Topics set:       topic-a, topic-b, topic-c
  Description:      <one-line description>
  Homepage:         <url or "—">
  Commit:           <short-sha>
  Push verified:    yes
  Repo URL:         https://github.com/<owner>/<repo>
```

After the summary, append any captured warnings from Phases F and G on their own lines. Each warning carries a retry command so the user can act without re-deriving the right gh invocation. Possible warnings:

- `WARN_TOPICS` — topics PUT failed, or round-trip showed a mismatch
- `WARN_DESCRIPTION` — description sync failed
- `WARN_HOMEPAGE` — homepage sync failed (only when sync was actually attempted)
- `WARN_PUSH` — local HEAD ≠ remote branch tip after push

If all four are clean, omit the warning section entirely.

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
| Topics round-trip mismatch | Continue; warn at end with expected vs. actual list. |
| Description sync fails | Continue; warn at end with retry command. |
| Homepage sync fails | Continue; warn at end with retry command. |
| GitHub Pages not enabled, or `homepageUrl` already set | Skip homepage sync silently. |
| Push rejected | Stop; instruct `git pull --rebase`. |
| Push SHA verification mismatch | Continue; warn at end with local vs. remote SHA. |
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
