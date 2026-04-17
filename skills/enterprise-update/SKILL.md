---
name: enterprise-update
version: 2.1.0
description: Pull and apply skill updates from tobias-nf/ironclaw-skills on GitHub. Supports scheduled auto-check.
activation:
  keywords: [update skills, check for updates, skill updates, what's new, install skill, setup updates]
  patterns: ["(?i)check.*(skill).*(update|version)", "(?i)update.*(skill)", "(?i)(install|add).*(new skill)", "(?i)setup.*(update|skill)"]
  tags: [skills, updates, maintenance]
  max_context_tokens: 2000
---

# Skill Updater

Fetch skill updates from `https://github.com/tobias-nf/ironclaw-skills` (branch: `main`, path: `skills/`) and apply them.

**Source repo:** `tobias-nf/ironclaw-skills`
**Personal data — never overwrite:** `config.json`, `PERSONAL.md`

---

## Setup

When the user says "setup updates", "enable auto-updates", or first time this skill runs:

1. Ask **how often to check for updates**:
   - "Daily at 9am" → `0 9 * * *` (default)
   - "Twice a day" → `0 9,17 * * *`
   - "Weekly on Monday" → `0 9 * * 1`
   - Custom cron
2. Ask **auto-apply or notify only**:
   - **Notify only** (default) — message the user when updates are available, wait for approval
   - **Auto-apply** — automatically update skills, notify afterwards

### Create mission

Check `mission_list` first — skip if `skill-update-check` already exists.

**Notify only:**
```
mission_create(
  name: "skill-update-check",
  goal: "Check for skill updates. (1) Clone tobias-nf/ironclaw-skills to temp dir. (2) Compare version: in each SKILL.md against local installed versions. (3) If any updates or new skills are available, send a message listing them with ⬆ for updates and ✨ for new. (4) Clean up temp dir. Stay silent if everything is up to date.",
  cadence: "<user's chosen cron>"
)
```

**Auto-apply:**
```
mission_create(
  name: "skill-update-check",
  goal: "Check for and apply skill updates. (1) Clone tobias-nf/ironclaw-skills to temp dir. (2) Compare versions. (3) For each outdated skill: backup config.json and PERSONAL.md, rsync updated files. (4) For new skills: install if no unmet dependencies. (5) Send a message summarizing what was updated. (6) Clean up temp dir. Stay silent if everything is up to date.",
  cadence: "<user's chosen cron>"
)
```

### Confirm

> Skill update check configured:
> - **Schedule:** <frequency>
> - **Mode:** <notify only / auto-apply>
>
> Say **"check for updates"** anytime for a manual check.

---

## When to use

- User says "check for updates", "update skills", "what's new"
- Scheduled mission fires
- First-time install of a new skill from the repo

---

## Workflows

### 1. Status Check

```bash
WORKSPACE=<ironclaw workspace path>
TMPDIR=$(mktemp -d)

git clone --depth=1 --branch main https://github.com/tobias-nf/ironclaw-skills $TMPDIR

for skill_dir in $TMPDIR/skills/*/; do
  skill=$(basename $skill_dir)
  repo_version=$(grep -m1 '^version:' $skill_dir/SKILL.md 2>/dev/null | sed 's/version: *//' | tr -d '"')
  local_version=$(grep -m1 '^version:' $WORKSPACE/skills/$skill/SKILL.md 2>/dev/null | sed 's/version: *//' | tr -d '"' || echo "not installed")
  echo "$skill: local=$local_version repo=$repo_version"
done

rm -rf $TMPDIR
```

Display a table:

```
Skill                    Local     Repo      Status
tasks-standalone         1.0.0     1.0.0     ✓ up to date
tasks-standalone-digest  —         1.0.0     ✨ new
meetings-standalone      1.0.0     1.1.0     ⬆ update available
```

### 2. Update a Skill

#### Backup personal data
```bash
BACKUP_DIR=$WORKSPACE/.update-backups/$SKILL/$(date +%Y%m%dT%H%M%S)
mkdir -p $BACKUP_DIR
for f in config.json PERSONAL.md; do
  [ -f $WORKSPACE/skills/$SKILL/$f ] && cp $WORKSPACE/skills/$SKILL/$f $BACKUP_DIR/
done
```

#### Fetch and apply
```bash
TMPDIR=$(mktemp -d)
git clone --depth=1 --branch main https://github.com/tobias-nf/ironclaw-skills $TMPDIR

rsync -av \
  --exclude="config.json" \
  --exclude="PERSONAL.md" \
  $TMPDIR/skills/$SKILL/ \
  $WORKSPACE/skills/$SKILL/

rm -rf $TMPDIR
```

#### Report
```
✓ tasks-standalone: 1.0.0 → 1.1.0
✓ meetings-standalone: 1.0.0 → 1.1.0
⏭ enterprise-update: 2.1.0 (up to date)
✨ tasks-standalone-triage: installed (new)
```

### 3. Install New Skill

If the repo has a skill not installed locally:

1. Check `requires.skills` in frontmatter — ensure dependencies are installed first
2. Copy skill files to `$WORKSPACE/skills/<name>/`
3. If the skill has `credentials` in frontmatter, prompt user for API keys
4. Report: "Installed **<name>** v<version>"

### 4. Rollback

If an update fails:
```bash
cp -a $BACKUP_DIR/* $WORKSPACE/skills/$SKILL/
```

Tell the user what failed and what was restored.

---

## Available Skills

Skills in `tobias-nf/ironclaw-skills`:

| Skill | Description |
|-------|-------------|
| `tasks` | Task management with local-first sync |
| `tasks-standalone` | Task management, cloud-only via HTTP |
| `tasks-standalone-digest` | Morning task summary |
| `tasks-standalone-triage` | Task health check |
| `tasks-mcp` | Task management via MCP |
| `meetings` | Meeting transcript processing with local sync |
| `meetings-standalone` | Meeting transcript processing, cloud-only |
| `email-to-tasks` | Extract tasks from emails (paste or Gmail API) |
| `enterprise-update` | This skill — updates from the repo |

## Quick Reference

| Action | What to do |
|--------|-----------|
| Status check | Clone repo, compare `version:` in SKILL.md, clean up |
| Update skill | Backup personal data → rsync from clone → clean up |
| New skill | Check dependencies → copy → prompt for credentials |
| Rollback | Restore from `.update-backups/<skill>/<timestamp>/` |
| Setup auto-check | Create `skill-update-check` mission with chosen cron |
