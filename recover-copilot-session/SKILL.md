---
name: recover-copilot-session
description: Use when the user asks to recover, resume, continue, inspect, or reconstruct a previous GitHub Copilot session from ~/.copilot/session-state
---

# Recover Copilot Session

## Overview

Recover enough context from GitHub Copilot session state files to continue the user's work in the current thread. Copilot stores session state in `~/.copilot/session-state/`, where each subdirectory is named after a session ID and contains JSON state files. Treat these files as the source of historical facts, then verify the live workspace before taking action.

**Core principle:** separate confirmed session facts from inferred next steps. Do not pretend the old session state is current until you verify it.

## Session Storage

Copilot session data lives at `~/.copilot/session-state/`:

- Each subdirectory is named after a **session ID** (UUID format).
- Inside each directory there are JSON files with conversation history, workspace context, and agent state.
- The latest modified directory typically corresponds to the most recent session.

## Workflow

1. **Load local instructions first**
   - Read `~/AGENTS.md`.
   - Search for relevant `AGENTS.md` files above and below the current workspace, then apply the nearest rule first.

2. **Locate candidate session directories**
   - Prefer clues from the user: session id, date, task keyword, repository path, branch name, commit, error text, or command.
   - List session directories sorted by last write time to find the most recent ones.
   - Search inside JSON files for keywords related to the user's task.

   ```powershell
   # List recent session directories
   Get-ChildItem "$env:USERPROFILE\.copilot\session-state" -Directory | Sort-Object LastWriteTime -Descending | Select-Object Name, LastWriteTime
   ```

   ```powershell
   # Search for a keyword across all session state files
   Get-ChildItem "$env:USERPROFILE\.copilot\session-state" -Recurse -Filter *.json -File | Select-String -Pattern "keyword or session id" -List | ForEach-Object { $_.Path }
   ```

3. **Read the session state files**
   - Read the JSON files in the candidate session directory.
   - Identify the user's original goal and any later corrections.
   - Extract the latest meaningful assistant update, tool result, error, blocker, and verification status.
   - Note files, repositories, branches, worktrees, commands, services, ports, commits, tags, and deployment targets mentioned.
   - Ignore stale plan text when later messages or tool output supersede it.
   - Do not expose secrets, tokens, passwords, or private credentials found in logs.

4. **Produce a recovery summary**
   - Label confirmed facts as "From session log".
   - Label guesses or deductions as "Inference".
   - Include: goal, current known state, completed work, unfinished work, blockers, and safest next step.
   - Mention the exact session directory path used.

5. **Continue from the recovered state**
   - Re-check the live workspace before editing, testing, committing, or deploying.
   - If the next step is clear, proceed with the task using relevant skills and project rules.
   - If several plausible continuations remain, ask the smallest necessary question before acting.
   - If recovery is impossible, explain what was searched and what clue would narrow the search.

## Search Hints

PowerShell examples:

```powershell
# List sessions with details
$sessionRoot = "$env:USERPROFILE\.copilot\session-state"
Get-ChildItem $sessionRoot -Directory | Sort-Object LastWriteTime -Descending | Format-Table Name, LastWriteTime, @{N='Files';E={(Get-ChildItem $_.FullName -File).Count}}
```

```powershell
# Read all JSON files in a specific session
Get-ChildItem "$env:USERPROFILE\.copilot\session-state\<session-id>" -Filter *.json | ForEach-Object { Write-Host "=== $($_.Name) ===" -ForegroundColor Cyan; Get-Content $_.FullName | ConvertFrom-Json | ConvertTo-Json -Depth 10 }
```

```powershell
# Search for text across all session state files
Get-ChildItem "$env:USERPROFILE\.copilot\session-state" -Recurse -Include "*.json" | Select-String -Pattern "search-term" | Select-Object Filename, LineNumber, Line
```

## Common Mistakes

- Stopping after finding a session directory without extracting the latest actionable state.
- Treating old tool output as current truth without re-running a cheap live check.
- Summarizing every message instead of focusing on the user's goal, state, blockers, verification, and next step.
- Continuing from an inferred plan while ignoring a later user correction in the same session.
- Exposing secrets or credentials found in session state files.
