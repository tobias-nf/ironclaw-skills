---
name: taskboard-triage
version: "1.0.0"
description: Review task health — flag overdue, stale, and blocked tasks. Suggest follow-ups and resolutions. Cloud-only.
activation:
  keywords:
    - triage tasks
    - review tasks
    - task health
    - overdue tasks
    - stale tasks
    - blocked tasks
    - follow up
    - what needs attention
  patterns:
    - "(?i)(triage|review|audit) (my )?(tasks|work|backlog)"
    - "(?i)what (needs|requires) (attention|action|follow.?up)"
    - "(?i)(overdue|stale|stuck|blocked) tasks"
  tags:
    - task-management
    - triage
    - health-check
  max_context_tokens: 1500
requires:
  skills: [taskboard]
credentials:
  - name: taskboard_api_key
    provider: taskboard
    location:
      type: bearer
    hosts:
      - "taskboard.commitment-tracker-aiops-sandbox.site"
      - "dev.taskboard.commitment-tracker-aiops-sandbox.site"
---

# Task Triage — Cloud-Only

Review task health, flag problems, and suggest actions. Used both in-conversation (user asks "triage my tasks") and by the `taskboard-triage` mission for scheduled runs.

## Gathering data

Fetch tasks from the cloud:
```
http(method="GET", url="{base_url}/api/v1/tasks/me?limit=200")
http(method="GET", url="{base_url}/api/v1/tasks/me/owed?limit=200")
```

For each flagged task, get recent activity to check staleness:
```
http(method="GET", url="{base_url}/api/v1/tasks/{task_id}/activity")
```

## Triage checks

Run these checks against all open tasks (status not in completed, failed, cancelled):

### 1. Overdue
Tasks where `deadline` is in the past.
- Flag: "**<title>** (<task_id>) — overdue by <N> days, status: <status>"
- Suggest: "Mark as completed, extend deadline, or escalate?"

### 2. Stale (no updates)
Tasks in `in_progress` where `updated_at` is more than 7 days ago.
- Verify by checking activity — if no activity entries in 7 days, it's stale.
- Flag: "**<title>** (<task_id>) — no updates in <N> days"
- Suggest: "Still working on this? Add a progress comment, or mark blocked."

### 3. Blocked without resolution
Tasks in `blocked` status for more than 3 days.
- Flag: "**<title>** (<task_id>) — blocked for <N> days"
- Suggest: "What's blocking this? Can I help unblock or escalate?"

### 4. Pending too long
Tasks in `pending` for more than 5 days (never started).
- Flag: "**<title>** (<task_id>) — pending for <N> days, never started"
- Suggest: "Start working on this, delegate, or deprioritize?"

### 5. Waiting on others (owed to me)
Tasks owed to the user where the assignee hasn't updated in 3+ days.
- Flag: "**<title>** (<task_id>) — waiting on <agent>, no update in <N> days"
- Suggest: "Follow up with <agent>? I can draft a message."

### 6. Priority mismatch
Tasks marked `emergency` or `urgent` that have been open for more than their expected window:
- Emergency: open > 1 day
- Urgent: open > 3 days
- Flag: "**<title>** (<task_id>) — <priority> priority but open for <N> days"
- Suggest: "Downgrade priority or take action?"

## Presenting results

```
## Task Triage — <today's date>

### Needs Immediate Attention
- **<title>** (<task_id>) — <reason>
  → <suggestion>

### Follow-Up Recommended
- **<title>** (<task_id>) — <reason>
  → <suggestion>

### Stale / Consider Closing
- **<title>** (<task_id>) — <reason>
  → <suggestion>

---
**Summary:** <N> items need attention out of <total> open tasks.
Want me to take action on any of these?
```

**Rules:**
- Omit empty sections
- If nothing is flagged: "All tasks look healthy. No action needed."
- Group by severity: immediate attention first, then follow-ups, then stale
- For each item, provide a concrete suggestion the user can approve
- If user approves a suggestion (e.g. "yes, follow up with Alice"), execute it:
  - Draft follow-up → add as comment on the task
  - Extend deadline → update via API
  - Mark completed → update status
  - Escalate → add comment noting escalation

## Delivery

- **In conversation:** Display the triage inline, wait for user to act
- **From mission:** Send via `message` tool. Only send if there are items flagged — silent when all healthy.
- **Tone:** Direct and actionable — problems + suggestions, not descriptions
