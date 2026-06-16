---
name: recover-zcode-session
description: Use when the user asks to recover, resume, continue, inspect, or reconstruct a previous ZCode session from ~/.zcode
---

# Recover ZCode Session

## Overview

Recover enough context from ZCode session logs to continue the user's work in the current thread. ZCode stores session data in two locations: model I/O JSONL files and an application log. Treat these as the source of historical facts, then verify the live workspace before taking action.

**Core principle:** separate confirmed session facts from inferred next steps. Do not pretend the old session state is current until you verify it.

## Data Sources

| Source | Path | Format |
|--------|------|--------|
| Model I/O records | `~/.zcode/cli/rollout/model-io-<sessionId>.jsonl` | JSONL, one request/response pair per line |
| Today's app log | `~/.zcode/v2/logs/YYYY-MM-DD.log` | Structured text log |
| Tasks index | `~/.zcode/v2/tasks-index.sqlite` | SQLite (task metadata only) |

### Model I/O File Structure

Each line is a JSON object with these fields:

```json
{
  "type": "model_io",
  "sessionId": "sess_xxx",
  "turnId": "turn_xxx",
  "traceId": "trace_xxx",
  "requestId": "uuid",
  "querySource": "main_turn | session_title | ...",
  "startedAt": "2026-06-16T09:12:20.935Z",
  "completedAt": "2026-06-16T09:12:35.856Z",
  "durationMs": 14921,
  "attempt": 1,
  "model": { "modelId": "GLM-5.2", "providerId": "builtin:zai-start-plan", "role": "main", "source": "session" },
  "request": { "body": { "model": "GLM-5.2", "messages": [...], "tools": [...], ... } },
  "response": { "text": "...", "toolCalls": [...], "finishReason": "tool-calls|stop", "usage": {...} }
}
```

Key fields in the response:
- `response.text` — assistant's text reply
- `response.toolCalls` — array of tool calls made
- `response.finishReason` — `"tool-calls"` (tool used) or `"stop"` (final reply)
- `response.usage` — token usage info

## Workflow

1. **Load local instructions first**
   - Read `~/AGENTS.md`.
   - Search for relevant `AGENTS.md` files above and below the current workspace, then apply the nearest rule first.

2. **Locate the session**
   - Prefer clues from the user: session id, date, task keyword, repository path, or error text.
   - If a session ID is given, look for `~/.zcode/cli/rollout/model-io-<sessionId>.jsonl`.
   - If a date is given, search `~/.zcode/v2/logs/YYYY-MM-DD.log` for the session ID.
   - If no date is given, search the app log for today.
   - Search the tasks index SQLite to find sessions by workspace or title:
     ```sql
     SELECT task_id, title, workspace_path, provider, model, created_at, updated_at
     FROM tasks
     WHERE workspace_path LIKE '%keyword%'
        OR title LIKE '%keyword%'
        OR task_id = 'sess_xxx'
     ORDER BY updated_at DESC;
     ```
   - Use `rg` or `findstr` to search for keywords across all model-io files:
     ```powershell
     findstr /M "keyword" "$env:USERPROFILE\.zcode\cli\rollout\model-io-*.jsonl"
     ```
     ```powershell
     rg --files-with-matches "keyword" "$env:USERPROFILE\.zcode\cli\rollout\"
     ```

3. **Read the matching model-io file carefully**
   - Each JSONL line is one model request/response pair (a "turn").
   - Parse all lines in order to reconstruct the conversation flow:
     ```powershell
     # Check total turns
     findstr /R /C:"^{" "$env:USERPROFILE\.zcode\cli\rollout\model-io-sess_xxx.jsonl" | find /C /V ""
     
     # Extract all assistant text responses
     python -c "import json; f=open('C:/Users/Admin/.zcode/cli/rollout/model-io-sess_xxx.jsonl'); [print(json.loads(l)['response'].get('text','')[:200]) for l in f if l.strip()]"
     ```
   - Identify the user's original goal and any later corrections.
   - Extract the latest meaningful assistant update, tool result, error, blocker, and verification status.
   - Note files, repositories, branches, worktrees, commands, services, ports, commits, tags, and deployment targets mentioned in the log.
   - Ignore stale plan text when later messages or tool output supersede it.
   - Do not expose secrets, tokens, passwords, or private credentials found in logs.

4. **Also check the app log for session lifecycle events**
   - Search the app log for session events (creation, resume, completion):
     ```powershell
     findstr "sess_xxx" "$env:USERPROFILE\.zcode\v2\logs\YYYY-MM-DD.log"
     ```
   - Key events to look for: `session/create`, `session/resume`, `sendPrompt`, `task_complete`, `repo snapshot upload accepted`

5. **Produce a recovery summary**
   - Label confirmed facts as "From session log".
   - Label guesses or deductions as "Inference".
   - Include: goal, current known state, completed work, unfinished work, blockers, and safest next step.
   - Mention the exact session log path used.

6. **Continue from the recovered state**
   - Re-check the live workspace before editing, testing, committing, or deploying.
   - If the next step is clear, proceed with the task using the relevant skills and project rules.
   - If several plausible continuations remain, ask the smallest necessary question before acting.
   - If recovery is impossible, explain what was searched and what clue would narrow the search.

## Batch Processing Script (PowerShell)

```powershell
# Extract all assistant text from every turn
python -c "
import json, sys
path = r'$env:USERPROFILE\.zcode\cli\rollout\model-io-sess_xxx.jsonl'
with open(path) as f:
    for i, line in enumerate(f):
        if not line.strip(): continue
        rec = json.loads(line)
        text = rec.get('response', {}).get('text', '') or ''
        tools = rec.get('response', {}).get('toolCalls', []) or []
        finish = rec.get('response', {}).get('finishReason', '')
        duration = rec.get('durationMs', 0)
        print(f'--- Turn {i} | finish={finish} | {duration}ms ---')
        if text:
            print(f'  Asst: {text[:300]}')
        if tools:
            names = [t.get('function',{}).get('name','?') for t in tools[:5]]
            print(f'  Tools({len(tools)}): {names}')
        print()
"
```

## Search Hints

- Model I/O files are in `~/.zcode/cli/rollout/` with naming pattern `model-io-<sessionId>.jsonl`.
- App logs are in `~/.zcode/v2/logs/` with naming pattern `YYYY-MM-DD.log`.
- The tasks index SQLite is at `~/.zcode/v2/tasks-index.sqlite`.
- Session snapshots may also be found in `~/.zcode/v2/tasks-index.sqlite` under the `tasks` table.
- When searching for a session by keyword, use `rg` across all model-io files first, as they contain the actual conversation content.
- If the session was very long (many turns), the model-io file can be quite large (multiple MB).

## Common Mistakes

- Stopping after finding a session path without extracting the latest actionable state.
- Treating old tool output as current truth without re-running a cheap live check.
- Reading only the first turn instead of scanning all turns to find the latest state.
- Forgetting that `querySource` helps distinguish user turns (`main_turn`) from auto-generated turns (`session_title`).
- Summarizing every message instead of focusing on the user's goal, state, blockers, verification, and next step.
- Continuing from an inferred plan while ignoring a later user correction in the same session.
