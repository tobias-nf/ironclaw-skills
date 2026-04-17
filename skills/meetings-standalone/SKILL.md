---
name: taskboard-meetings-standalone
version: "1.0.0"
description: Process meeting transcripts — extract action items, deduplicate against cloud tasks, create new tasks after approval. Cloud-only, no local files.
activation:
  keywords: [meeting, transcript, meeting notes, action items, meeting summary, process meeting, meeting recap]
  patterns: ["(?i)process.*(meeting|transcript)", "(?i)(meeting|call).*(notes|recap|summary)", "(?i)extract.*(action items|tasks).*(meeting|transcript)", "(?i)(here is|here's|I have).*(transcript|meeting notes)"]
  tags: [meetings, transcripts, action-items]
  max_context_tokens: 2000
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
    setup_instructions: "Same API key as the taskboard-standalone skill."
---

# Meetings Standalone — Transcript to Tasks (Cloud-Only)

Parses meeting transcripts, extracts action items, deduplicates against existing Taskboard tasks via the REST API, and creates new tasks after user approval. No local files — everything goes directly to the cloud.

## When to Use

- User pastes a meeting transcript or attaches a transcript file
- User says "process this meeting", "extract action items from this transcript"
- A transcript arrives from any source (Fireflies, Otter, Google Meet, manual notes)

## Workflow

### Step 1: Get the transcript

- **Pasted text:** collect all messages from the user
- **File attachment:** read the file
- **Multi-part messages:** wait for all parts, concatenate, then process as one
- **Ask for title + date** if not obvious from the transcript

### Step 2: Gather existing tasks for dedup

Fetch from the cloud:
```
http(method="GET", url="{base_url}/api/v1/tasks/me?limit=200")
http(method="GET", url="{base_url}/api/v1/tasks/me/owed?limit=200")
```

Also fetch the agent list for matching people to agents:
```
http(method="GET", url="{base_url}/api/v1/agents/me/assignable")
```

And get the current user's identity:
```
http(method="GET", url="{base_url}/api/v1/agents/me")
```

### Step 3: Parse the transcript

Extract structured action items. Include the current task list and agent list in your parsing context so you can flag matches and resolve assignees.

Output JSON:
```json
{
  "title": "Meeting title",
  "date": "YYYY-MM-DD",
  "attendees": ["Name1", "Name2"],
  "summary": "Brief 2-4 sentence summary of what was discussed",
  "action_items": [
    {
      "title": "Short task title (max 80 chars)",
      "context": "2-4 sentences of WHY — what was discussed, decisions made, background",
      "owner_type": "mine | waiting_on | irrelevant",
      "assigned_to": "agent-id or null",
      "owed_to": "agent-id or null",
      "related_person": "Name of counterpart (not the user)",
      "tags": ["tag1", "tag2"],
      "priority": "low | standard | urgent | emergency",
      "due_date": "YYYY-MM-DD or null",
      "existing_task_id": "T-2026-XXXXX or null",
      "match_reason": "Why this matches an existing task, or null"
    }
  ]
}
```

**Extraction rules:**

- **`context`**: capture the WHY, not just a restatement of the title. Include what was discussed, what decision was made, who raised it.
- **`owner_type`**:
  - `mine` — the user must do this
  - `waiting_on` — someone else owes this to the user
  - `irrelevant` — not actionable for the user, skip
- **`assigned_to`**: match attendee names/emails to agent IDs from the assignable agents list. If no match, leave null.
- **`owed_to`**: for `waiting_on` items, set to the user's agent ID.
- **`person_resolved`**: true if the related_person matched a Taskboard agent, false if external/unknown.
- **`existing_task_id`**: if a current task covers the same work, set its ID to update instead of duplicating
- **`match_reason`**: explain why it matches (e.g. "Existing task T-2026-00042 already tracks the partnership agreement review")
- **`priority`**: "By end of week" → urgent. "When you get a chance" → low. Default: standard.
- **`tags`**: 1-3 reusable labels (e.g. "legal", "engineering", "compliance")

### Step 4: Present summary for approval

Show the user a summary before creating anything:

```
## Meeting: <title> (<date>)

**Attendees:** Name1, Name2, Name3

**Summary:** <2-4 sentences>

### Action Items

#### New Tasks
1. **<title>** — assigned to <agent>, priority: <priority>, due: <date>
   _<context>_

2. **<title>** — waiting on <person> (Taskboard agent), owed to you
   _<context>_

3. **Waiting: <person> to <deliver thing>** — waiting on <person> (external, not in Taskboard), assigned to you
   _<context>_

#### Updates to Existing Tasks
3. **<existing title>** (T-2026-XXXXX) — add context from this meeting
   _Match: <match_reason>_
   _New context: <context>_

#### Skipped
- <irrelevant item> (not actionable for you)

---
Approve all? Or tell me which to change/skip. (e.g. "skip 2, change 1 priority to urgent")
```

**Wait for user approval.** Do NOT create or push anything until the user confirms.

### Step 5: Process approved items

After approval, for each approved action item:

**New tasks (`existing_task_id` is null):**

**Case 1: `owner_type: mine` or `waiting_on` with resolved agent:**
1. Create task:
```
http(method="POST", url="{base_url}/api/v1/tasks", body={
  "title": "...",
  "description": "**From meeting:** <meeting title> (<date>)\n\n<context>",
  "assigned_to": "<agent-id>",
  "priority": "<priority>",
  "deadline": "<due_date>T00:00:00Z",
  "tags": ["meeting-action-item", ...],
  "visibility": "private"
})
```
2. If `waiting_on` with resolved agent, add owed_to:
```
http(method="POST", url="{base_url}/api/v1/tasks/{task_id}/owed-to", body={
  "agent_id": "<user's agent-id>"
})
```

**Case 2: `waiting_on` with unresolved person (not in Taskboard):**
1. Create task assigned to the user (you are tracking it):
```
http(method="POST", url="{base_url}/api/v1/tasks", body={
  "title": "Waiting: <person> to <deliver thing>",
  "description": "**From meeting:** <meeting title> (<date>)\n\n<context>\n\n**Waiting on:** <person> (external — not a Taskboard agent)",
  "assigned_to": "<user's agent-id>",
  "priority": "<priority>",
  "deadline": "<due_date>T00:00:00Z",
  "tags": ["meeting-action-item", "waiting-on"],
  "visibility": "private"
})
```
No owed_to call needed — the task is yours, tracking someone external.

**For both cases**, add meeting reference:
```
http(method="POST", url="{base_url}/api/v1/tasks/{task_id}/references", body={
  "type": "origin",
  "source": "meeting",
  "title": "<meeting title> — <date>"
})
```

**Updates to existing tasks (`existing_task_id` is set):**
1. Update description with meeting context:
```
http(method="PATCH", url="{base_url}/api/v1/tasks/{existing_task_id}", body={
  "description": "<existing description>\n\n**Meeting update:** <meeting title> (<date>)\n<context>"
})
```
2. Update priority/deadline if the meeting changed them:
```
http(method="PATCH", url="{base_url}/api/v1/tasks/{existing_task_id}", body={
  "priority": "<new priority>",
  "deadline": "<new deadline>"
})
```

### Step 6: Confirm

```
Meeting processed:
- **N new tasks** created in Taskboard
- **N existing tasks** updated with meeting context
- **N items** skipped

All tasks are visible in the Taskboard dashboard.
```

---

## Edge Cases

- **No action items found:** "No action items detected in this transcript. Want me to look again with different criteria?"
- **All items match existing tasks:** "All action items are already tracked. I've added meeting context to N existing tasks."
- **Unknown attendees for `mine` tasks:** "Could not match 'Sarah' to a Taskboard agent. Created task unassigned."
- **Unknown attendees for `waiting_on` tasks:** Created as "Waiting: Sarah to ..." assigned to you with `waiting-on` tag. Shows up in your task list and triage.
- **Ambiguous ownership:** Default to `owner_type: mine` and let the user reassign.

## Personal Extensions

If `PERSONAL.md` exists in this skill's directory, read and apply those instructions as additions or overrides.
