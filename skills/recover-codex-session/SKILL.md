---
name: recover-codex-session
description: Use when the user asks to recover, resume, continue, inspect, or reconstruct a previous Codex session from ~/.codex/sessions
---

# Recover Codex Session

## Overview

Recover enough context from Codex session logs to continue the user's work in the current thread. Treat `~/.codex/sessions` as the source of historical facts, then verify the live workspace before taking action.

**Core principle:** separate confirmed session facts from inferred next steps. Do not pretend the old session state is current until you verify it.

## Workflow

1. **Load local instructions first**
   - Read `~/AGENTS.md`.
   - Search for relevant `AGENTS.md` files above and below the current workspace, then apply the nearest rule first.
   - If a relevant `.mcp.json` exists, try to load or account for it before continuing.

2. **Locate candidate session logs**
   - Prefer clues from the user: session id, rollout id, date, task keyword, repository path, branch name, commit, error text, or command.
   - If the user gives a date, search `~/.codex/sessions/YYYY/MM/DD` first.
   - If no date is given, start with today's directory: `~/.codex/sessions/YYYY/MM/DD`.
   - If the first pass fails, broaden to all of `~/.codex/sessions`.
   - Use read-only commands. Prefer `rg` for text search and file listing; fall back to shell-native search when needed.

3. **Read the matching `.jsonl` carefully**
   - Identify the user's original goal and any later corrections.
   - Extract the latest meaningful assistant update, tool result, error, blocker, and verification status.
   - Note files, repositories, branches, worktrees, commands, services, ports, commits, tags, and deployment targets mentioned in the log.
   - Ignore stale plan text when later messages or tool output supersede it.
   - Do not expose secrets, tokens, passwords, or private credentials found in logs.

4. **Produce a recovery summary**
   - Label confirmed facts as "From session log".
   - Label guesses or deductions as "Inference".
   - Include: goal, current known state, completed work, unfinished work, blockers, and safest next step.
   - Mention the exact session log path used.

5. **Continue from the recovered state**
   - Re-check the live workspace before editing, testing, committing, or deploying.
   - If the next step is clear, proceed with the task using the relevant skills and project rules.
   - If several plausible continuations remain, ask the smallest necessary question before acting.
   - If recovery is impossible, explain what was searched and what clue would narrow the search.

## Search Hints

PowerShell examples:

```powershell
$today = Get-Date -Format 'yyyy\\MM\\dd'
$sessionRoot = Join-Path $HOME ".codex\\sessions"
$todayDir = Join-Path $sessionRoot $today
Get-ChildItem -LiteralPath $todayDir -Filter *.jsonl -File -ErrorAction SilentlyContinue | Sort-Object LastWriteTime -Descending | Select-Object FullName, LastWriteTime, Length
```

```powershell
rg --line-number --fixed-strings "keyword or session id" "$HOME\\.codex\\sessions"
```

When `rg` is unavailable:

```powershell
Get-ChildItem -LiteralPath "$HOME\\.codex\\sessions" -Recurse -Filter *.jsonl -File | Select-String -Pattern "keyword or session id"
```

## Common Mistakes

- Stopping after finding a log path without extracting the latest actionable state.
- Treating old tool output as current truth without re-running a cheap live check.
- Summarizing every message instead of focusing on the user's goal, state, blockers, verification, and next step.
- Continuing from an inferred plan while ignoring a later user correction in the same log.
