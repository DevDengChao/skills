---
name: recover-opencode-session
description: Use when the user asks to recover, resume, continue, inspect, or reconstruct a previous opencode session from the local SQLite database
---

# Recover opencode Session

## Overview

Recover enough context from opencode session logs to continue the user's work in the current thread. opencode stores session data in a local SQLite database at `~/.local/share/opencode/opencode.db`. Treat this database as the source of historical facts, then verify the live workspace before taking action.

**Core principle:** separate confirmed session facts from inferred next steps. Do not pretend the old session state is current until you verify it.

## Database Schema

The opencode database (`~/.local/share/opencode/opencode.db`) contains these key tables:

| Table | Description |
|-------|-------------|
| `session` | Session metadata (id, title, directory, timestamps, model, agent, cost) |
| `message` | Messages grouped by session (role: `user` / `assistant`, stored as JSON in `data` column) |
| `part` | Individual content parts of a message (type: `text`, `reasoning`, `tool`, `step-start`, `step-finish`) |
| `session_message` | System-level events (type: `agent-switched`, `model-switched`) |

### Key Fields

- **Timestamps**: stored as Unix epoch **milliseconds** (divide by 1000 to get seconds for `datetime()`)
- **Content**: message body and part data are JSON stored in the `data` text column
- **Sequencing**: `message.time_created` orders messages; within a message, `part.time_created` orders parts

## Workflow

1. **Load local instructions first**
   - Read the project's `AGENTS.md` or equivalent.
   - Search for relevant instruction files above and below the current workspace, then apply the nearest rule first.

2. **Locate candidate sessions**
   - Prefer clues from the user: session id, date, task keyword, repository path, branch name, commit, error text, or command.
   - Query the `session` table to find matching sessions.

   ```sql
   -- Find sessions by date range (past N days)
   SELECT id, title, directory,
          datetime(time_created/1000, 'unixepoch', 'localtime') AS created,
          datetime(time_updated/1000, 'unixepoch', 'localtime') AS updated
   FROM session
   WHERE time_created >= 1000 * strftime('%s', 'now', '-2 days', 'localtime')
   ORDER BY time_created DESC;

   -- Find sessions by keyword in title
   SELECT id, title, directory,
          datetime(time_created/1000, 'unixepoch', 'localtime') AS created
   FROM session
   WHERE title LIKE '%keyword%'
   ORDER BY time_created DESC;

   -- Find sessions by project directory
   SELECT id, title,
          datetime(time_created/1000, 'unixepoch', 'localtime') AS created
   FROM session
   WHERE directory LIKE '%nuan-xiang%'
   ORDER BY time_created DESC;
   ```

3. **Read the session messages**
   - Messages are stored in the `message` table with `role` (user/assistant) in the JSON `data` column.
   - Actual content is in the `part` table, joined via `message_id`.

   ```sql
   -- Get all user messages with text content
   SELECT m.time_created, json_extract(p.data, '$.text') AS content
   FROM message m
   JOIN part p ON p.message_id = m.id
   WHERE m.session_id = 'ses_xxx'
     AND json_extract(m.data, '$.role') = 'user'
     AND json_extract(p.data, '$.type') = 'text'
   ORDER BY m.time_created;

   -- Get all assistant tool calls
   SELECT m.time_created,
          json_extract(p.data, '$.tool') AS tool,
          json_extract(p.data, '$.state.status') AS status,
          substr(json_extract(p.data, '$.state.output'), 1, 200) AS output_preview
   FROM message m
   JOIN part p ON p.message_id = m.id
   WHERE m.session_id = 'ses_xxx'
     AND json_extract(m.data, '$.role') = 'assistant'
     AND json_extract(p.data, '$.type') = 'tool'
   ORDER BY m.time_created, p.time_created;
   ```

4. **Extract the recovery summary**
   - Identify the user's original goal and any later corrections.
   - Extract the latest meaningful assistant update, tool result, error, blocker, and verification status.
   - Note files, repositories, branches, worktrees, commands, services, ports, commits, tags, and deployment targets mentioned in the log.
   - Ignore stale plan text when later messages or tool output supersede it.
   - Do not expose secrets, tokens, passwords, or private credentials found in logs.

5. **Produce a recovery summary**
   - Label confirmed facts as "From session log".
   - Label guesses or deductions as "Inference".
   - Include: goal, current known state, completed work, unfinished work, blockers, and safest next step.
   - Mention the exact session id used.

6. **Continue from the recovered state**
   - Re-check the live workspace before editing, testing, committing, or deploying.
   - If the next step is clear, proceed with the task using the relevant skills and project rules.
   - If several plausible continuations remain, ask the smallest necessary question before acting.
   - If recovery is impossible, explain what was searched and what clue would narrow the search.

## Query Examples (PowerShell)

```powershell
# Connect and list recent sessions
sqlite3 "$env:USERPROFILE\.local\share\opencode\opencode.db" "SELECT id, title, directory, datetime(time_created/1000,'unixepoch','localtime') AS created FROM session ORDER BY time_created DESC LIMIT 10;"

# Find sessions by keyword
sqlite3 "$env:USERPROFILE\.local\share\opencode\opencode.db" "SELECT id, title, datetime(time_created/1000,'unixepoch','localtime') AS created FROM session WHERE title LIKE '%your-keyword%' ORDER BY time_created DESC;"

# Export all text content from a session
sqlite3 "$env:USERPROFILE\.local\share\opencode\opencode.db" "SELECT m.time_created, json_extract(m.data,'$.role') AS role, json_extract(p.data,'$.type') AS part_type, CASE WHEN json_extract(p.data,'$.type')='text' THEN json_extract(p.data,'$.text') WHEN json_extract(p.data,'$.type')='reasoning' THEN '[reasoning]' WHEN json_extract(p.data,'$.type')='tool' THEN '[tool: '||json_extract(p.data,'$.tool')||']' ELSE json_extract(p.data,'$.type') END AS content FROM message m JOIN part p ON p.message_id=m.id WHERE m.session_id='ses_xxx' ORDER BY m.time_created, p.time_created;"

# Get session summary with cost info
sqlite3 "$env:USERPROFILE\.local\share\opencode\opencode.db" "SELECT id, title, directory, datetime(time_created/1000,'unixepoch','localtime') AS created, datetime(time_updated/1000,'unixepoch','localtime') AS updated, cost, tokens_input, tokens_output, agent, model FROM session WHERE id='ses_xxx';"
```

## Search Hints

When the SQLite database is the starting point:

- If the user gives a date, filter `time_created` relative to that date (remember: milliseconds).
- If no date is given, check the last 2 days, then broaden if needed.
- Search by `title` first (most visible), then by `directory` (project), then by content in `part.data` (if needed).
- The `session.directory` field indicates which project the session belongs to.

## Common Mistakes

- Stopping after finding a session id without extracting the latest actionable state.
- Treating old tool output as current truth without re-running a cheap live check.
- Forgetting that timestamps are in **milliseconds** (need `/1000` for `datetime()`).
- Summarizing every message instead of focusing on the user's goal, state, blockers, verification, and next step.
- Continuing from an inferred plan while ignoring a later user correction in the same session.
- Exposing secrets or credentials found in tool outputs or database content.
