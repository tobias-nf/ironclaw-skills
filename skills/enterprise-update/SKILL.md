---
name: enterprise-update
version: 1.1.0
description: Pull and apply ironclaw skill updates from GitHub. The agent IS the update mechanism — reads SKILL.md frontmatter for version info, preserves personal data, and applies changes intelligently.
activation:
  keywords: [update skills, check for updates, skill updates, what's new, update enterprise skills, install skill]
  patterns: ["(?i)check.*(skill|enterprise).*(update|version)", "(?i)update.*(skill|enterprise)", "(?i)(install|add).*(new skill)"]
  tags: [skills, updates, maintenance]
  max_context_tokens: 2000
---

# enterprise-update

Fetch ironclaw skill updates from `https://github.com/nearai/ironclaw` (branch: `staging`, path: `skills/`) and apply them intelligently. The agent IS the update mechanism — no Python sync scripts needed.

**Personal data files — never overwrite:** `config.json`, `PERSONAL.md`

---

## When to use this skill

- User says "check for skill updates", "update my skills", "what's new in the skills"
- Scheduled or manual update check
- First-time install of a new skill that appears in the repo

---

## File Format

Each skill has a single `SKILL.md` containing all metadata and instructions. Version is in the YAML frontmatter:

```yaml
---
name: tasks
version: 2.1.0
---
```

---

## Workflows

### 1. Status Check (what needs updating?)

Fetch the repo into a temp directory, then compare versions:

```bash
WORKSPACE=<ironclaw workspace path>
TMPDIR=$(mktemp -d)

# Fetch latest skills
git clone --depth=1 --branch staging https://github.com/nearai/ironclaw $TMPDIR

# Compare versions
for skill_dir in $TMPDIR/skills/*/; do
  skill=$(basename $skill_dir)
  repo_version=$(grep -m1 '^version:' $skill_dir/SKILL.md 2>/dev/null | sed 's/version: *//')
  local_version=$(grep -m1 '^version:' $WORKSPACE/skills/$skill/SKILL.md 2>/dev/null | sed 's/version: *//' || echo "not installed")
  echo "$skill: local=$local_version repo=$repo_version"
done

rm -rf $TMPDIR
```

Display a table showing each skill with local vs repo version. Mark outdated skills with `⬆` and uninstalled ones with `✨ new`.

---

### 2. Update a Skill

#### Step 1: Fetch latest

```bash
TMPDIR=$(mktemp -d)
git clone --depth=1 --branch staging https://github.com/nearai/ironclaw $TMPDIR
```

#### Step 2: Check if update is needed

Parse `version:` from `$TMPDIR/skills/<name>/SKILL.md` and compare to the local copy.

If repo version ≤ local version: skip (already up to date).

#### Step 3: Backup personal data

Always back up personal data BEFORE touching the skill directory:

```bash
BACKUP_DIR=$WORKSPACE/.update-backups/$SKILL/$(date +%Y%m%dT%H%M%S)
mkdir -p $BACKUP_DIR

for f in config.json PERSONAL.md; do
  [ -f $WORKSPACE/skills/$SKILL/$f ] && cp $WORKSPACE/skills/$SKILL/$f $BACKUP_DIR/
done
```

#### Step 4: Copy updated files

```bash
rsync -av \
  --exclude="config.json" \
  --exclude="PERSONAL.md" \
  $TMPDIR/skills/$SKILL/ \
  $WORKSPACE/skills/$SKILL/

rm -rf $TMPDIR
```

#### Step 5: Report

```
✓ tasks: 2.0.0 → 2.1.0
✓ meetings: 2.0.0 → 2.1.0
⏭ enterprise-update: 1.1.0 (up to date)
```

---

### 3. First Install of a New Skill

If the repo has a skill not installed locally:

1. Fetch repo as above
2. Check `requires.skills` in the frontmatter — ensure dependencies are installed first
3. Copy skill files to `$WORKSPACE/skills/<name>/`
4. If the skill has a `credentials` section in the frontmatter, prompt the user for any required API keys

---

### 4. Rollback

If an update fails after the backup:

```bash
cp -a $BACKUP_DIR/* $WORKSPACE/skills/$SKILL/
```

Tell the user what failed and what was restored.

---

## Quick Reference

| Action | What to do |
|--------|-----------|
| Status check | Clone repo to tmp, compare `version:` in SKILL.md frontmatter, clean up |
| Update skill | Backup personal data → rsync from tmp clone → clean up |
| New skill | Check `requires.skills` → copy files → prompt for credentials |
| Rollback | Restore from `.update-backups/<skill>/<timestamp>/` |
