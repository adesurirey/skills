---
name: add-skill
description: Move an already-authored agent skill (typically just created via Cursor's built-in `create-skill`) into your skills repo, then optionally commit, push, and install it via `npx skills`. Resolves the source skill from a path or name (searches `.cursor/skills/` and `~/.cursor/skills/`), moves the directory into `skills/<name>/`, and walks through commit/push/install. Runnable from any cwd. Use when the user says "add-skill", "ingest skill", "promote skill", or wants to import a freshly-created skill into their skills repo.
argument-hint: "[skill-name-or-path]"
allowed-tools: AskQuestion, Read, Write, Glob, Grep, Bash(ls:*), Bash(mv:*), Bash(mkdir:*), Bash(realpath:*), Bash(readlink:*), Bash(test:*), Bash(printenv:*), Bash(git status:*), Bash(git diff:*), Bash(git add:*), Bash(git commit:*), Bash(git push:*), Bash(git rev-parse:*), Bash(git remote:*), Bash(git -C:*), Bash(npx skills:*)
---

## Constants

- **TARGET_REPO**: resolved from the `$SKILLS_REPO` environment variable (absolute local path to your skills repo). See Step 1.
- **Source search paths** (in order):
  1. `<cwd>/.cursor/skills/<name>/`
  2. `~/.cursor/skills/<name>/`
  3. `~/.claude/skills/<name>/` (fallback if the user authored with Claude Code)

## Arguments

Parse `$ARGUMENTS` as: `[skill-name-or-path]`

- **skill-name-or-path** (optional): either a skill name (kebab-case slug — will be searched in the standard locations above) or an absolute/relative path to a directory containing a `SKILL.md`. If omitted, ask in Step 2.

## Step 1 — Locate the target repo

Read `$SKILLS_REPO` from the environment (`printenv SKILLS_REPO`). Expand `~` if present, then resolve to an absolute path. Verify it is a git repo containing a `skills/` directory:

```bash
test -d "$SKILLS_REPO/skills" && test -d "$SKILLS_REPO/.git"
```

If `$SKILLS_REPO` is unset, empty, or doesn't pass the check, ask the user once (this prompt is the per-invocation fallback — do not persist):

> Where is your skills repo? (absolute path)

Re-verify and stop if it's still not a valid skills repo, suggesting the user `export SKILLS_REPO=…` in their shell rc to avoid the prompt next time.

From here on, refer to it as `$TARGET`. All `git` operations against it use `git -C "$TARGET"`.

## Step 2 — Locate the source skill

### 2a. Resolve the argument

If `$ARGUMENTS` contains a `/` or starts with `~`/`.`/`/`, treat it as a path:
- Expand `~`, then resolve to absolute (`realpath`).
- Verify the directory contains a `SKILL.md`. If not, stop and report.

Otherwise treat it as a **name** and search the source paths from the **Constants** section in order. Collect all matches.

If no argument was provided, ask:

> Source skill? Reply with a name (e.g. `release-notes`) or an absolute path to a skill directory.

Then resolve as above.

### 2b. Disambiguate

- **No matches** → report all paths searched and ask the user for an explicit path.
- **One match** → use it.
- **Multiple matches** → use `AskQuestion` listing each candidate path; let the user pick.

### 2c. Validate the source

Run `realpath` on the resolved source to follow symlinks. Then:

- Read `SKILL.md` and extract the `name:` from frontmatter. This is the authoritative skill name (it may differ from the directory name).
- **Already-in-repo guard**: if the resolved real path is inside `$TARGET`, stop with "Source is already inside the target repo at `<rel-path>` — nothing to do."
- **Symlink to repo guard**: if the source is a symlink whose target is inside `$TARGET` (i.e. `npx skills` already installed it from this repo), stop with the same message and suggest the user wanted a different skill.
- **Name collision**: if `$TARGET/skills/<source-name>/` already exists, report it and stop. (Layout is flat — one directory per skill, directory name matches frontmatter `name:`.)

## Step 3 — Preview

Print a short summary before mutating anything:

```
Source:      <abs-source-path>
Destination: $TARGET/skills/<source-name>/
Skill name:  <source-name>
```

Then `AskQuestion`:

> Move now?

Options: `Yes` / `No`. Stop on `No`.

## Step 4 — Move the skill

Destination is `$TARGET/skills/<source-name>/`.

1. Refuse to overwrite: if `$DEST` already exists, stop and ask the user to resolve it manually.
2. Move with `mv "$SRC" "$DEST"` — a plain `mv` is fine because the source is typically not tracked by git in `$TARGET`. If the move crosses filesystems and fails, fall back to `cp -R "$SRC" "$DEST" && rm -rf "$SRC"`.
3. Verify: `test -f "$DEST/SKILL.md"`.

## Step 5 — Review gate

Print the destination path and the first ~10 lines of the moved `SKILL.md` (frontmatter). Ask:

> Open `skills/<source-name>/SKILL.md` and review. Reply **commit** to proceed, or describe changes you'd like first.

Loop on edit requests (`Read`/`Write` against the moved file) until the user replies `commit` (or `ship it`, `lgtm`, `proceed`). Do not advance without explicit confirmation.

## Step 6 — Commit

All git ops use `git -C "$TARGET" …`:

1. `git -C "$TARGET" status` — confirm the new directory is the only change (warn if there are unrelated edits in `$TARGET`).
2. `git -C "$TARGET" add "skills/<source-name>"`.
3. `git -C "$TARGET" commit -m "Add <source-name> skill"` — match the repo's short imperative log style. No body, no AI/co-author trailers.

If `$TARGET`'s current branch isn't `main`, ask whether to commit on the current branch or `git -C "$TARGET" checkout main` first.

## Step 7 — Push?

Check for an origin remote: `git -C "$TARGET" remote get-url origin`. Capture its value as `$REMOTE_URL` — this is also used to derive the install slug in Step 8.

- **No `origin`** → report this, skip the push, remember "not pushed" for Step 8.
- **`origin` exists** → `AskQuestion`:

  > Push to origin?

  Options: `Yes` / `No`. If `Yes`, run `git -C "$TARGET" push` (use `-u origin <branch>` only if upstream isn't set). Report the remote URL on success.

## Step 8 — Install?

Use `AskQuestion`:

> Install `<source-name>` globally via `npx skills`?

Options depend on Step 7:
- If pushed → `Yes — from remote` / `Yes — from local checkout` / `No`.
- Else → `Yes — from local checkout` / `No`.

Derive the remote slug from `$REMOTE_URL` (Step 7). Examples:
- `git@github.com:owner/repo.git` → `owner/repo`
- `https://github.com/owner/repo.git` → `owner/repo`

Strip a trailing `.git` and any leading `git@host:` / `https://host/` prefix.

Commands (run from any cwd — `$TARGET` is absolute):
- Remote: `npx skills add <slug> --skill <source-name> -g -y`
- Local: `npx skills add "$TARGET" --skill <source-name> -g -y`

Run and report the output verbatim. After install, remind the user that the new skill is picked up in **new** chat sessions only — the current one won't see it until reload.

## Step 9 — Output

Print a single-line summary, omitting segments that didn't happen:

```
Added <source-name> — skills/<source-name>/SKILL.md
  moved from <abs-source> | committed | pushed | installed (global, from <remote|local>)
```
