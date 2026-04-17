# IronClaw Skills for Taskboard

[IronClaw](https://github.com/nearai/ironclaw) skills for task management via the [Taskboard](https://github.com/tobias-nf/taskboard) API.

## Skills

| Skill | Description |
|-------|-------------|
| `tasks` | Core task management — create, update, list, digest, triage |
| `tasks-digest` | Morning task summary with follow-up suggestions |
| `tasks-triage` | Health check — overdue, stale, blocked, priority mismatches |
| `meetings` | Meeting transcript → deduplicated tasks with approval |
| `email-to-tasks` | Extract tasks from emails (paste or Gmail API) |
| `bug-report` | Create bug reports for IronClaw devs with chat history |
| `skill-updater` | Auto-update skills from this repo |

## Relationship to IronClaw

These skills replace the built-in commitment tracking skills in IronClaw:

- [`commitment-setup`](https://github.com/nearai/ironclaw/tree/staging/skills/commitment-setup)
- [`commitment-triage`](https://github.com/nearai/ironclaw/tree/staging/skills/commitment-triage)
- [`commitment-digest`](https://github.com/nearai/ironclaw/tree/staging/skills/commitment-digest)

Remove the commitment-* skills after installing these.

See all IronClaw skills: https://github.com/nearai/ironclaw/tree/staging/skills

## Install

Copy skills to your IronClaw skills directory:
```bash
for skill in tasks tasks-digest tasks-triage meetings email-to-tasks bug-report skill-updater; do
  mkdir -p /path/to/ironclaw/skills/$skill
  cp skills/$skill/SKILL.md /path/to/ironclaw/skills/$skill/SKILL.md
done
```

Then say "setup taskboard" to configure the API connection.

## API Key

Get your key from the Taskboard dashboard (Settings) or ask an admin.
Format: `hive_sk_<agent-id>_<secret>`
