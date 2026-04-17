---
name: enterprise-update
version: 2.0.0
description: Pull and apply skill updates from tobias-nf/ironclaw-skills on GitHub.
activation:
  keywords: [update skills, check for updates, skill updates, what's new, install skill]
  patterns: ["(?i)check.*(skill).*(update|version)", "(?i)update.*(skill)", "(?i)(install|add).*(new skill)"]
  tags: [skills, updates, maintenance]
  max_context_tokens: 2000
---

# Skill Updater

Fetch skill updates from `https://github.com/tobias-nf/ironclaw-skills` (branch: `main`, path: `skills/`) and apply them.

**Source repo:** `tobias-nf/ironclaw-skills`
**Personal data — never overwrite:** `config.json`, `PERSONAL.md`

---

## When to use

- User says "check for updates", "update skills", "what's new"
- First-time install of a skill from the repo

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
⏭ enterprise-update: 2.0.0 (up to date)
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
| `enterprise-update` | This skill — updates from the repo |

## Quick Reference

| Action | What to do |
|--------|-----------|
| Status check | Clone repo, compare `version:` in SKILL.md, clean up |
| Update skill | Backup personal data → rsync from clone → clean up |
| New skill | Check dependencies → copy → prompt for credentials |
| Rollback | Restore from `.update-backups/<skill>/<timestamp>/` |
