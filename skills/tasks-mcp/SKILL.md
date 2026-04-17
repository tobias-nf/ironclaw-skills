---
name: taskboard-mcp
version: "1.0.0"
description: Task management via Taskboard MCP server. Cloud-only, no local files. For testing MCP integration.
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
    - mcp
    - personal-assistant
  max_context_tokens: 2000
---

# Taskboard MCP — Cloud-Only Task Management

Manages tasks entirely through the Taskboard MCP server. No local files — all reads and writes go directly to the cloud API via MCP tools.

**Requires:** Taskboard MCP server connected via:
```bash
ironclaw mcp add taskboard https://taskboard.commitment-tracker-aiops-sandbox.site/mcp/sse
ironclaw mcp auth taskboard   # enter your hive_sk_* API key
```

All task operations use MCP tools. The tools are auto-discovered from the server — call them by name with the documented parameters.

---

## Setup

When the user says "setup taskboard" or this skill activates for the first time:

1. Check if the MCP server is connected: try calling `task_list_mine`. If it fails, tell the user:
   > Taskboard MCP server is not connected. Run:
   > ```
   > ironclaw mcp add taskboard https://taskboard.commitment-tracker-aiops-sandbox.site/mcp/sse
   > ironclaw mcp auth taskboard
   > ```
   > Then try again.

2. Verify identity: call `agent_whoami`. Show the user their agent ID and email.

3. Confirm: "Taskboard connected as **<agent-id>**. I'll track tasks from our conversations."

---

## Mode A: Passive Signal Detection

When the user says something implying an obligation but is NOT explicitly asking to track it.

**Triggers:** "I need to...", "I promised Sarah...", "The report is due Friday"

**Action:** Create a task directly in the cloud:
```
task_create(
  title: "<concise title>",
  description: "<context from conversation>",
  priority: "<low|standard|urgent|emergency>",
  deadline: "<ISO8601 or omit>",
  tags: ["auto-detected"]
)
```

After creation, briefly note: "I've tracked a task: **<title>** (<task_id>)"

**Priority inference:**
- `emergency`: due today or production impact
- `urgent`: due within 3 days, named-person ask
- `standard`: due within 2 weeks or no hard deadline
- `low`: no deadline, whenever

## Mode B: Explicit Task Creation

User says: "track this", "create a task", "I need to do X by Friday".

**Action:**
```
task_create(
  title: "...",
  description: "...",
  priority: "standard",
  deadline: "2026-04-25T00:00:00Z",
  assigned_to: "agent-id",
  parent_id: "T-2026-XXXXX",
  visibility: "public",
  tags: ["tag1", "tag2"]
)
```

To find who to assign to: `agent_assignable` returns active agents.

Confirm: "Created: **<title>** (<task_id>), priority: <priority>, due: <date>"

## Mode C: Task Updates and Resolution

User says: "done with X", "mark T-2026-00042 done", "I'm blocked on the review".

**Find the task:**
- If user gives a task ID → `task_get(task_id: "T-2026-XXXXX")`
- If user describes it → `task_list_mine` and search by title

**Update:**
```
task_update(
  task_id: "T-2026-XXXXX",
  status: "completed"
)
```

**Add a comment:**
```
task_comment(
  task_id: "T-2026-XXXXX",
  body: "Completed — reviewed and sent to legal."
)
```

Confirm: "Resolved: **<title>** (<task_id>)"

**Valid status transitions:**
- pending → in_progress, completed, blocked, cancelled
- in_progress → blocked, review, completed, failed, cancelled
- blocked → in_progress, cancelled
- review → completed, in_progress, cancelled

## Mode D: Task Digest

User asks: "show my tasks", "what's overdue?", "what's on my plate?"

**Gather data:**
```
task_list_mine()          → tasks assigned to me
task_list_owed()          → tasks others owe me
task_list_created()       → tasks I created for others
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

### Waiting On Others
- **<title>** (<task_id>) — assigned to <agent>, <status>

---
Did I miss anything?
```

Omit empty sections. Zero tasks: "No open tasks. You're clear."

## Mode E: Task Detail

User asks about a specific task: "tell me about T-2026-00042", "what's the status of the review task?"

**Gather:**
```
task_get(task_id: "T-2026-XXXXX")
task_get_activity(task_id: "T-2026-XXXXX")
task_get_tags(task_id: "T-2026-XXXXX")
task_get_owed_to(task_id: "T-2026-XXXXX")
```

Present a summary with status, priority, assignee, deadline, recent activity, and subtasks (if parent_id references exist).

## Mode F: Browse and Search

User asks: "show all tasks", "what tasks are blocked?", "tasks tagged legal"

```
task_list_visible(status: "blocked")
task_list_visible(tag: "legal")
task_list_visible(assigned_to: "alice")
```

Filter params: status (comma-separated), priority, tag, assigned_to, limit, offset.

---

## MCP Tools Reference

These tools are auto-discovered from the Taskboard MCP server. Listed here for reference.

### Task Tools

| Tool | Description |
|------|-------------|
| `task_create` | Create task. Params: title (required), description, assigned_to, priority, deadline, parent_id, visibility, tags[] |
| `task_get` | Get task by ID. Params: task_id |
| `task_list_mine` | My assigned tasks. Params: status, priority, tag |
| `task_list_created` | Tasks I created. Params: status |
| `task_list_owed` | Tasks owed to me. Params: status |
| `task_list_visible` | All visible tasks. Params: status, priority, tag, assigned_to, limit, offset |
| `task_update` | Update task. Params: task_id, status, priority, title, description, assigned_to, parent_id, visibility |
| `task_cancel` | Cancel task. Params: task_id |
| `task_comment` | Add comment. Params: task_id, body |
| `task_get_activity` | Get activity timeline. Params: task_id |

### Tag Tools

| Tool | Description |
|------|-------------|
| `tag_list` | List all tags |
| `tag_create` | Create tag. Params: name, color |
| `task_add_tag` | Add tag to task. Params: task_id, tag_name |
| `task_remove_tag` | Remove tag. Params: task_id, tag_id |
| `task_get_tags` | List tags on task. Params: task_id |

### Stakeholder Tools

| Tool | Description |
|------|-------------|
| `task_add_owed_to` | Add stakeholder. Params: task_id, agent_id |
| `task_remove_owed_to` | Remove stakeholder. Params: task_id, agent_id |
| `task_get_owed_to` | List stakeholders. Params: task_id |
| `task_add_mention` | Add mention. Params: task_id, agent_id |
| `task_remove_mention` | Remove mention. Params: task_id, agent_id |
| `task_get_mentions` | List mentions. Params: task_id |

### Reference Tools

| Tool | Description |
|------|-------------|
| `task_add_reference` | Add reference. Params: task_id, ref_type (origin/related/blocks/depends_on/output), source, title, url, external_id |
| `task_list_references` | List references. Params: task_id |
| `task_delete_reference` | Delete reference. Params: task_id, reference_id |

### Agent Tools

| Tool | Description |
|------|-------------|
| `agent_whoami` | Current agent profile |
| `agent_assignable` | Active agents for assignment |
| `agent_list` | List all agents. Params: search, type, active, limit, offset |
| `agent_get` | Get agent. Params: agent_id |

### Task Fields

- **Statuses:** draft, pending, in_progress, blocked, review, completed, failed, cancelled
- **Priorities:** low, standard, urgent, emergency
- **Visibility:** public (default), private
- **Subtasks:** set parent_id when creating. Visibility inherited from parent.
