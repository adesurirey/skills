# skills

Personal collection of [agent skills](https://www.skills.sh) for use with Cursor, Claude Code, Codex, and any other [agent supported by `skills.sh`](https://github.com/vercel-labs/skills#supported-agents).

Each skill lives in `skills/<name>/SKILL.md` and follows the [Agent Skills specification](https://agentskills.io).

## Install

Using the [`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
# Install everything globally (available across all projects)
npx skills add adesurirey/skills -g

# Install a single skill
npx skills add adesurirey/skills --skill add-skill -g

# Target specific agents
npx skills add adesurirey/skills -g -a cursor -a claude-code
```

## Configure

`add-skill` reads the local path to your skills repo from `$SKILLS_REPO`. Add to your shell rc:

```bash
export SKILLS_REPO="$HOME/code/<you>/skills"
```

If unset, `add-skill` will prompt you for the path each invocation.

## Adding a new skill

### Recommended: author anywhere, then `add-skill`

This repo ships an `add-skill` skill that automates the "promote into the repo" half of the workflow. Once you've installed it globally (`npx skills add adesurirey/skills --skill add-skill -g`), the flow from any project is:

1. Author the skill anywhere your agent puts new skills — typically `~/.cursor/skills/<name>/`, `~/.claude/skills/<name>/`, or `<project>/.cursor/skills/<name>/`. Use your agent's built-in skill creator (e.g. Cursor's `create-skill`) or hand-roll a `SKILL.md`.
2. Invoke `add-skill` (with the skill name or path as an argument). It will:
   - locate the source skill,
   - move it into `$SKILLS_REPO/skills/<name>/`,
   - gate on review, then commit,
   - optionally push and re-install globally.

See [`skills/add-skill/SKILL.md`](./skills/add-skill/SKILL.md) for the full step-by-step.

### Manual scaffold

If you'd rather scaffold a blank skill directly in the repo:

```bash
cd skills
npx skills init my-new-skill
```

Then edit the generated `SKILL.md`, commit, and push.
