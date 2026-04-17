---
name: email-to-tasks
version: "1.0.0"
description: Extract tasks and action items from emails. Supports pasted emails, Gmail API, and periodic inbox scanning.
activation:
  keywords:
    - email
    - inbox
    - check email
    - scan email
    - email tasks
    - process email
    - email action items
    - gmail
    - unread emails
  patterns:
    - "(?i)(check|scan|process|review) (my )?(email|inbox|gmail)"
    - "(?i)(extract|find|get) (tasks|action items) from (email|inbox)"
    - "(?i)(here is|here's|I got) (an )?email"
    - "(?i)email from .+"
  tags:
    - email
    - task-management
    - inbox
  max_context_tokens: 2000
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
    setup_instructions: "Same API key as the taskboard skill."
  - name: google_token
    provider: google
    location:
      type: bearer
    hosts:
      - "gmail.googleapis.com"
    oauth:
      authorization_url: "https://accounts.google.com/o/oauth2/v2/auth"
      token_url: "https://oauth2.googleapis.com/token"
      scopes:
        - "https://www.googleapis.com/auth/gmail.readonly"
      refresh:
        strategy: refresh_token
    setup_instructions: "Connect your Google account to read emails. Only read-only access is requested."
---

# Email to Tasks

Extract tasks and action items from emails and create them in Taskboard. Works with pasted emails, Gmail API, or periodic inbox scanning.

## Setup

When the user says "setup email tasks" or "connect my email":

1. Ask which **email source** to use:
   - **Gmail API** (recommended) — scans inbox automatically
   - **Paste only** — user pastes emails manually, no API needed
2. If Gmail API: authenticate via the credential system (OAuth with gmail.readonly scope)
3. Verify Gmail access:
   ```
   http(method="GET", url="https://gmail.googleapis.com/gmail/v1/users/me/profile")
   ```
   If successful → "Connected to Gmail as **<email>**"
4. Ask **how often to scan** (only if Gmail API):
   - "Every hour" → `0 * * * *`
   - "Every 2 hours during work hours" → `0 9,11,13,15,17 * * 1-5` (default)
   - "Morning only" → `0 8 * * 1-5`
5. Ask for **scan criteria**:
   - Check unread only? (default: yes)
   - Filter by labels? (default: INBOX)
   - Skip newsletters/automated? (default: yes)

### Create mission (Gmail API only)

```
mission_create(
  name: "email-scan",
  goal: "Scan Gmail for actionable emails. (1) http(method='GET', url='https://gmail.googleapis.com/gmail/v1/users/me/messages?q=is:unread+label:inbox&maxResults=20') to get unread messages. (2) For each message, http(method='GET', url='https://gmail.googleapis.com/gmail/v1/users/me/messages/{id}?format=full') to read it. (3) Extract action items — skip newsletters, automated notifications, and FYI-only emails. (4) For actionable emails, check against existing tasks in Taskboard to avoid duplicates. (5) Present extracted items to user for approval before creating tasks. If no actionable items found, stay silent.",
  cadence: "<user's chosen cron>"
)
```

### Confirm

> Email scanning ready.
> - **Source:** Gmail (<email>)
> - **Scan frequency:** <schedule>
> - **Criteria:** unread inbox messages, skipping newsletters
>
> Say **"check email"** anytime for a manual scan.

---

## Mode A: Manual — User Pastes an Email

User pastes email text or says "here's an email from Sarah about the budget review".

### Step 1: Parse the email

Extract:
```json
{
  "from": "sender name <email>",
  "to": "recipient",
  "date": "YYYY-MM-DD",
  "subject": "Email subject",
  "action_items": [
    {
      "title": "Short task title (max 80 chars)",
      "context": "2-4 sentences — what was asked, why, any deadline mentioned",
      "owner_type": "mine | waiting_on | irrelevant",
      "related_person": "sender or person mentioned",
      "priority": "low | standard | urgent | emergency",
      "due_date": "YYYY-MM-DD or null",
      "existing_task_id": "T-2026-XXXXX or null"
    }
  ]
}
```

**Extraction rules:**
- "Can you review..." / "Please send..." / "We need..." → `mine`
- "I'll send you..." / "I'll get back to you..." / "We'll provide..." → `waiting_on`
- FYI / newsletter / no action → `irrelevant`
- Deadline cues: "by Friday", "end of week", "ASAP" → set due_date and priority accordingly
- Check against existing Taskboard tasks for dedup

### Step 2: Check for duplicates

```
http(method="GET", url="{base_url}/api/v1/tasks/me?limit=200")
http(method="GET", url="{base_url}/api/v1/tasks/me/owed?limit=200")
```

Compare extracted items against existing tasks by title similarity and context.

### Step 3: Present for approval

```
## Email: <subject> (from <sender>, <date>)

### Action Items

1. **<title>** — <owner_type>, priority: <priority>, due: <date>
   _<context>_

2. **Waiting: <person> to <thing>** — waiting on <person>
   _<context>_

#### Skipped
- <irrelevant item>

---
Approve all? Or tell me which to change/skip.
```

**Wait for approval before creating anything.**

### Step 4: Create tasks

For approved items, same logic as meetings skill:

**`mine` or `waiting_on` with resolved agent:**
```
http(method="POST", url="{base_url}/api/v1/tasks", body={
  "title": "...",
  "description": "**From email:** <subject> (from <sender>, <date>)\n\n<context>",
  "assigned_to": "<agent-id or user-id>",
  "priority": "<priority>",
  "deadline": "<due_date>T00:00:00Z",
  "tags": ["email-action-item"],
  "visibility": "private"
})
```

**`waiting_on` with unresolved person:**
```
http(method="POST", url="{base_url}/api/v1/tasks", body={
  "title": "Waiting: <person> to <thing>",
  "description": "**From email:** <subject> (from <sender>, <date>)\n\n<context>\n\n**Waiting on:** <person> (external)",
  "assigned_to": "<user's agent-id>",
  "tags": ["email-action-item", "waiting-on"],
  "visibility": "private"
})
```

Add email reference:
```
http(method="POST", url="{base_url}/api/v1/tasks/{task_id}/references", body={
  "type": "origin",
  "source": "email",
  "title": "<subject> — from <sender>, <date>"
})
```

---

## Mode B: Gmail API Scan

Triggered by "check email", "scan inbox", or by the email-scan mission.

### Step 1: Fetch unread messages

```
http(method="GET", url="https://gmail.googleapis.com/gmail/v1/users/me/messages?q=is:unread+label:inbox&maxResults=20")
→ {"messages": [{"id": "...", "threadId": "..."}], "resultSizeEstimate": N}
```

If no messages → "No unread emails. Inbox clear." Stop.

### Step 2: Read each message

For each message ID:
```
http(method="GET", url="https://gmail.googleapis.com/gmail/v1/users/me/messages/{id}?format=full")
```

Extract from headers: `From`, `To`, `Subject`, `Date`.
Extract body from `payload.parts` (prefer `text/plain`, fall back to `text/html`).

### Step 3: Filter

Skip emails that are:
- Newsletters (List-Unsubscribe header present)
- Automated notifications (noreply@, notifications@, etc.)
- Calendar invites (Content-Type: text/calendar)
- Marketing (common patterns: "unsubscribe", bulk sender headers)

### Step 4: Extract action items

For each remaining email, extract action items using the same rules as Mode A.

### Step 5: Deduplicate

Fetch existing tasks and compare. Skip items that match an existing task.

### Step 6: Present for approval

Group all extracted items across emails:

```
## Email Scan — <today's date>

**Scanned:** <N> unread emails, <M> actionable

### From: <sender> — <subject>
1. **<title>** — <owner_type>, priority: <priority>
   _<context>_

### From: <sender> — <subject>
2. **<title>** — <owner_type>, priority: <priority>
   _<context>_

---
Approve all? Or tell me which to change/skip.
```

**Wait for approval.** Only create tasks after user confirms.

### Step 7: Create tasks

Same as Mode A Step 4.

---

## Edge Cases

- **No action items in email:** "No action items found in this email. It's informational."
- **All duplicates:** "All items from this email are already tracked."
- **Gmail API unavailable:** Fall back to paste mode — "Gmail connection failed. You can paste the email text instead."
- **Long email threads:** Focus on the most recent message in the thread. Older context is background.
- **Forwarded emails:** "Please handle this" with a forwarded email → the action is in the forwarded content, not the forwarding message.

## Personal Extensions

If `PERSONAL.md` exists in this skill's directory, read and apply those instructions as additions or overrides.
