---
name: taskboard-standalone-digest
version: "1.0.0"
description: Compose and deliver task summaries from Taskboard API. Cloud-only — no local files.
activation:
  keywords:
    - show tasks
    - task digest
    - task summary
    - task report
    - my tasks
    - what do I owe
    - pending tasks
    - what's overdue
    - what's due
    - task status
    - what's on my plate
  patterns:
    - "(?i)(show|list|summarize|review) (my )?(tasks|obligations|deadlines)"
    - "(?i)what('s| is| are) (pending|overdue|due|open)"
    - "(?i)task (digest|report|status|summary)"
  tags:
    - task-management
    - digest
    - reporting
  max_context_tokens: 1500
requires:
  skills: [taskboard-standalone]
credentials:
  - name: taskboard_api_key
    provider: taskboard
    location:
      type: bearer
    hosts:
      - "taskboard.commitment-tracker-aiops-sandbox.site"
      - "dev.taskboard.commitment-tracker-aiops-sandbox.site"
---

# Task Digest — Cloud-Only

Compose a summary of the user's tasks from the Taskboard API. Used both in-conversation (user asks "show my tasks") and by the `taskboard-digest` mission for scheduled delivery.

## Gathering data

Fetch all relevant tasks from the cloud:

```
http(method="GET", url="{base_url}/api/v1/tasks/me?limit=200")
http(method="GET", url="{base_url}/api/v1/tasks/me/owed?limit=200")
http(method="GET", url="{base_url}/api/v1/tasks/me/created?limit=200")
http(method="GET", url="{base_url}/api/v1/agents/me")
```

Read `base_url` from the taskboard-standalone skill's stored config.

## Composing the digest

Group tasks and present in this order:

```
## Tasks — <today's date>

### Overdue / Emergency
- **<title>** (<task_id>) — due <date>, <status>, assigned to <agent>
  → Overdue by <N> days

### Due This Week
- **<title>** (<task_id>) — due <date>, <status>

### In Progress
- **<title>** (<task_id>) — started <date>

### Blocked
- **<title>** (<task_id>) — blocked since <date>
  → Follow up needed

### Pending (not started)
- **<title>** (<task_id>) — priority: <priority>

### Waiting On Others (owed to me)
- **<title>** (<task_id>) — assigned to <agent>, <status>
  → (follow-up suggested) if last updated 3+ days ago

### Tasks I Created (assigned to others)
- **<title>** (<task_id>) — assigned to <agent>, <status>

### Recently Completed
- <title> (<task_id>) — completed <date>

---
**Summary:** <N> active, <N> overdue, <N> waiting on others
Did I miss anything?
```

**Rules:**
- Omit empty sections entirely
- Keep each item to one line plus optional suggestion
- If zero tasks: "No open tasks. You're clear."
- Flag tasks past deadline as overdue with days count
- For blocked tasks, suggest follow-up
- For tasks assigned to others with no update in 3+ days, suggest follow-up
- Always end with "Did I miss anything?"

## Delivery

- **In conversation:** Display the digest inline
- **From mission:** Send via `message` tool to the user's preferred channel
- **Tone:** Concise and scannable — a dashboard, not a narrative
