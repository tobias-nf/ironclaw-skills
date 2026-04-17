---
name: taskboard-standalone
version: "1.0.0"
description: Task management via Taskboard REST API. Cloud-only, no local files, no sync. Simple and direct.
activation:
  keywords:
    - need to
    - have to
    - must do
    - promised
    - committed to
    - deadline
    - by friday
    - by tomorrow
    - follow up
    - get back to
    - remind me
    - track this
    - mark done
    - my tasks
    - what's overdue
    - task
    - taskboard
    - action item
    - blocked
    - subtask
  patterns:
    - "(?i)I (need|have|should|must|ought) to"
    - "(?i)(remind me|don't let me forget|make sure I)"
    - "(?i)(by|before|until) (monday|tuesday|wednesday|thursday|friday|saturday|sunday|tomorrow|tonight|end of)"
    - "(?i)(promised|committed|agreed) (to|that)"
    - "(?i)(slack|email|dm|text) message from .+: .+"
  tags:
    - task-management
    - personal-assistant
  max_context_tokens: 2000
credentials:
  - name: taskboard_api_key
    provider: taskboard
    location:
      type: bearer
    hosts:
      - "taskboard.commitment-tracker-aiops-sandbox.site"
      - "dev.taskboard.commitment-tracker-aiops-sandbox.site"
    setup_instructions: "Get your API key from Taskboard Settings or ask an admin. Format: hive_sk_<agent-id>_<secret>"
---

# Taskboard Standalone — Cloud-Only via HTTP

All task operations go directly to the Taskboard REST API. No local files, no sync, no workspace setup. Just HTTP calls.

## Setup

When the user says "setup taskboard" or this skill activates for the first time:

1. Ask for the **Taskboard URL**:
   - Production: `https://taskboard.commitment-tracker-aiops-sandbox.site` (default)
   - Dev: `https://dev.taskboard.commitment-tracker-aiops-sandbox.site`
   - Self-hosted: any URL
2. Ask for the **API key** (format: `hive_sk_<agent-id>_<secret>`)
3. Verify by calling:
   ```
   http(method="GET", url="{base_url}/api/v1/agents/me")
   ```
   If it returns the agent profile → "Connected as **<agent-id>**. Ready."
   If 401/403 → "API key is invalid. Check and try again."

Store the base URL for later use. The API key is managed by the credential system.

---

## Mode A: Passive Signal Detection

When the user says something implying an obligation but is NOT explicitly asking to track it.

**Triggers:** "I need to...", "I promised Sarah...", "The report is due Friday"

**Action:** Create a task directly:
```
http(method="POST", url="{base_url}/api/v1/tasks", body={
  "title": "<concise title>",
  "description": "<context from conversation>",
  "priority": "<low|standard|urgent|emergency>",
  "deadline": "<ISO8601 or omit>",
  "tags": ["auto-detected"]
})
```

After creation: "I've tracked a task: **<title>** (<task_id>)"

Do NOT interrupt conversation flow. Task creation is a side-effect.

**Priority inference:**
- `emergency`: due today, production impact
- `urgent`: within 3 days, named-person ask
- `standard`: within 2 weeks or no hard deadline (default)
- `low`: no deadline, whenever

## Mode B: Explicit Task Creation

User says: "track this", "create a task", "I need to do X by Friday".

**Action:**
```
http(method="POST", url="{base_url}/api/v1/tasks", body={
  "title": "...",
  "description": "...",
  "priority": "standard",
  "deadline": "2026-04-25T00:00:00Z",
  "assigned_to": "agent-id",
  "parent_id": "T-2026-XXXXX",
  "visibility": "public",
  "tags": ["tag1"]
})
```

To find assignees:
```
http(method="GET", url="{base_url}/api/v1/agents/me/assignable")
```

Confirm: "Created: **<title>** (<task_id>), priority: <priority>, due: <date>"

## Mode C: Task Updates and Resolution

User says: "done with X", "mark T-2026-00042 done", "I'm blocked".

**Find the task:**
- User gives task ID → get it directly
- User describes it → search my tasks:
  ```
  http(method="GET", url="{base_url}/api/v1/tasks/me")
  ```

**Update:**
```
http(method="PATCH", url="{base_url}/api/v1/tasks/{task_id}", body={
  "status": "completed"
})
```

**Add a comment:**
```
http(method="POST", url="{base_url}/api/v1/tasks/{task_id}/activity", body={
  "body": "Done — reviewed and sent to legal."
})
```

Confirm: "Resolved: **<title>** (<task_id>)"

**Valid status transitions:**
- pending → in_progress, completed, blocked, cancelled
- in_progress → blocked, review, completed, failed, cancelled
- blocked → in_progress, cancelled
- review → completed, in_progress, cancelled

## Mode D: Task Digest

User asks: "show my tasks", "what's overdue?", "what's on my plate?"

**Gather:**
```
http(method="GET", url="{base_url}/api/v1/tasks/me")
http(method="GET", url="{base_url}/api/v1/tasks/me/owed")
```

**Present grouped:**

```
## Tasks — <today's date>

### Overdue / Emergency
- **<title>** (<task_id>) — due <date>, <status>

### Due This Week
- **<title>** (<task_id>) — due <date>, <status>

### In Progress
- **<title>** (<task_id>) — <status>

### Pending
- **<title>** (<task_id>) — priority: <priority>

### Waiting On Others (owed to me)
- **<title>** (<task_id>) — assigned to <agent>, <status>

---
Did I miss anything?
```

Omit empty sections. Zero tasks: "No open tasks. You're clear."

## Mode E: Task Detail

User asks about a specific task: "tell me about T-2026-00042"

```
http(method="GET", url="{base_url}/api/v1/tasks/{task_id}")
http(method="GET", url="{base_url}/api/v1/tasks/{task_id}/activity")
http(method="GET", url="{base_url}/api/v1/tasks/{task_id}/tags")
```

Present: status, priority, assignee, deadline, description, recent activity.

## Mode F: Browse and Search

"show all tasks", "what tasks are blocked?", "tasks tagged legal"

```
http(method="GET", url="{base_url}/api/v1/tasks/visible?status=blocked")
http(method="GET", url="{base_url}/api/v1/tasks/visible?tag=legal")
http(method="GET", url="{base_url}/api/v1/tasks/visible?assigned_to=alice")
```

---

## Taskboard REST API Reference

Base URL: configured during setup.
Auth: credentials are injected automatically — never construct Authorization headers.

### Task Object

```json
{
  "id": "T-2026-00042",
  "title": "Review partnership agreement",
  "description": "Markdown description...",
  "created_by": "tobias.holenstein",
  "assigned_to": "alice",
  "visibility": "public",
  "status": "in_progress",
  "priority": "standard",
  "deadline": "2026-04-25T00:00:00Z",
  "parent_id": null,
  "created_at": "2026-04-15T08:00:00Z",
  "started_at": "2026-04-15T09:00:00Z",
  "completed_at": null,
  "updated_at": "2026-04-16T10:00:00Z"
}
```

**Statuses:** draft, pending, in_progress, blocked, review, completed, failed, cancelled
**Priorities:** low, standard, urgent, emergency
**Visibility:** public (default), private
**Subtasks:** set parent_id. Visibility inherited from parent.

### Agent Object

```json
{
  "id": "tobias.holenstein",
  "type": "user",
  "email": "tobias.holenstein@near.foundation",
  "slack_id": "U0AJ1HULDL3",
  "active": true
}
```

### Endpoints

#### Tasks

| Method | Path | Description |
|--------|------|-------------|
| POST | /tasks | Create task |
| GET | /tasks/me | My assigned tasks |
| GET | /tasks/me/created | Tasks I created |
| GET | /tasks/me/owed | Tasks owed to me |
| GET | /tasks/visible | All visible tasks |
| GET | /tasks/{id} | Get task detail |
| PATCH | /tasks/{id} | Update task |
| DELETE | /tasks/{id} | Cancel task |

**Create fields:** title (required), description, assigned_to, priority, deadline, parent_id, visibility, tags[]
**Update fields:** title, description, status, priority, deadline, assigned_to, parent_id, visibility
**Query params:** status (comma-separated), priority, tag, assigned_to, sort, limit, offset

#### Activity

| Method | Path | Description |
|--------|------|-------------|
| POST | /tasks/{id}/activity | Add comment: `{"body": "..."}` |
| GET | /tasks/{id}/activity | Get timeline |

#### Tags

| Method | Path | Description |
|--------|------|-------------|
| GET | /tags | List all tags |
| POST | /tags | Create: `{"name": "...", "color": "#hex"}` |
| GET | /tasks/{id}/tags | Tags on task |
| POST | /tasks/{id}/tags | Add: `{"tag_name": "..."}` |
| DELETE | /tasks/{id}/tags/{tagId} | Remove |

#### Stakeholders (owed_to)

| Method | Path | Description |
|--------|------|-------------|
| GET | /tasks/{id}/owed-to | List |
| POST | /tasks/{id}/owed-to | Add: `{"agent_id": "..."}` |
| DELETE | /tasks/{id}/owed-to/{agentId} | Remove |

#### Mentions

| Method | Path | Description |
|--------|------|-------------|
| GET | /tasks/{id}/mentions | List |
| POST | /tasks/{id}/mentions | Add: `{"agent_id": "..."}` |
| DELETE | /tasks/{id}/mentions/{agentId} | Remove |

#### References

| Method | Path | Description |
|--------|------|-------------|
| GET | /tasks/{id}/references | List |
| POST | /tasks/{id}/references | Add: `{"type": "related", "source": "slack", "title": "...", "url": "..."}` |
| DELETE | /tasks/{id}/references/{refId} | Remove |

Reference types: origin, related, blocks, depends_on, output

#### Agents

| Method | Path | Description |
|--------|------|-------------|
| GET | /agents/me | Current agent |
| PATCH | /agents/me | Update: email, slack_id, preferred_tool |
| GET | /agents/me/assignable | Active agents for assignment |
| GET | /agents | List all |
| GET | /agents/{id} | Get detail |
