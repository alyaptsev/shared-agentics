# Claude Code Hooks Reference

A decision-oriented reference for designing hooks. Hooks are deterministic shell commands (or HTTP/prompt/agent handlers) that run at specific lifecycle points in Claude Code — they add guardrails, inject context, validate output, and enforce rules without consuming LLM tokens.

---

## 1. When to use a hook vs other building blocks

| Need | Use | Why |
|------|-----|-----|
| Deterministic validation (format, length, required fields) | **Hook** | Code is faster, cheaper, and more reliable than LLM |
| Context injection (reminders, project state, env info) | **Hook** (SessionStart) | Runs once at start, no user action needed |
| Blocking dangerous operations | **Hook** (PreToolUse, exit 2) | Prevents the action before it happens |
| Nuanced quality assessment | **Skill** | Needs LLM reasoning |
| Multi-step task coordination | **Workflow** | Needs agent orchestration |
| Automated post-processing | **Hook** (PostToolUse) | Runs after every tool call of a type |

## 2. Hook events — decision matrix

### Events that can BLOCK (exit code 2)

| Event | Matches on | Use when |
|-------|-----------|----------|
| **PreToolUse** | Tool name (regex) | Block dangerous commands, validate tool inputs before execution |
| **UserPromptSubmit** | — (always fires) | Reject prompts that violate rules, transform input |
| **PermissionRequest** | Tool name | Auto-allow or auto-deny specific permissions |
| **Stop** | — | Prevent Claude from stopping until conditions are met |
| **SubagentStop** | Agent type | Prevent subagent from finishing until quality check passes |
| **ConfigChange** | Config source | Block unauthorized config modifications |
| **WorktreeCreate** | — | Prevent worktree creation under certain conditions |

### Events for context injection

| Event | When it fires | What stdout does |
|-------|--------------|-----------------|
| **SessionStart** | Session begins (startup/resume/clear/compact) | Stdout added to Claude's context |
| **PreToolUse** | Before any tool call | `additionalContext` field injected |
| **PostToolUse** | After tool succeeds | `additionalContext` field injected |
| **InstructionsLoaded** | CLAUDE.md files loaded | Can transform instructions |

### Events for observation only (cannot block)

| Event | Use for |
|-------|---------|
| **PostToolUse** | Logging, notifications, post-processing |
| **PostToolUseFailure** | Error tracking, alerting |
| **Notification** | Custom notification handling |
| **SessionEnd** | Cleanup, logging |
| **PreCompact / PostCompact** | State preservation around compaction |

## 3. Handler types

| Type | What it does | When to use |
|------|-------------|-------------|
| **command** | Runs a shell script, reads stdin JSON, outputs to stdout | Most hooks — validation scripts, context injection |
| **http** | POST to a URL with JSON body | External services, webhooks, remote validation |
| **prompt** | Single LLM turn (Haiku by default) | Quick semantic checks — "is this command safe?" |
| **agent** | Multi-turn LLM agent | Complex validation requiring tool use |

**Default to `command`** — it's fastest, cheapest, and most predictable. Use `prompt`/`agent` types only when you need LLM reasoning in the hook itself.

## 4. Configuration structure

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "regex_pattern",
        "hooks": [
          {
            "type": "command",
            "command": "path/to/script.sh",
            "timeout": 600
          }
        ]
      }
    ]
  }
}
```

### Configuration locations (priority order)

| Location | Scope | Share in git? |
|----------|-------|---------------|
| Managed policy | Organization-wide | Admin-controlled |
| `~/.claude/settings.json` | All projects on this machine | No |
| `.claude/settings.json` | This project | Yes |
| `.claude/settings.local.json` | This project, personal | No (gitignored) |

## 5. Input/Output schemas

### Input (all hooks receive via stdin)

```json
{
  "session_id": "abc123",
  "cwd": "/project/root",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /important",
    "description": "Delete files"
  }
}
```

Key event-specific input fields:

| Event | Key fields |
|-------|-----------|
| SessionStart | `source` (startup/resume/clear/compact) |
| UserPromptSubmit | `prompt` (user's text) |
| PreToolUse / PostToolUse | `tool_name`, `tool_input` (tool-specific) |
| Notification | `notification_type` |
| Stop | `stop_hook_active` (check to avoid infinite loop) |
| StopFailure | `error_type`, `error_message` |

### Output (hook writes to stdout)

**For PreToolUse** (most powerful output):
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "Explanation",
    "updatedInput": { "command": "modified command" },
    "additionalContext": "Text injected into Claude's context"
  }
}
```

**For SessionStart**: plain text on stdout → added to Claude's context.

**For most other events**: `{ "decision": "allow|block", "reason": "..." }`

## 6. Matcher patterns

Matchers are **regex patterns** (case-sensitive). They filter when hooks fire.

```
"Bash"              → exact tool name
"Edit|Write"        → either tool
"mcp__github__.*"   → all GitHub MCP tools
"mcp__.*__write.*"  → any MCP write operation
"startup|resume"    → SessionStart on startup or resume
```

## 7. Common patterns

### Block dangerous bash commands
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "node scripts/check-dangerous-command.js"
      }]
    }]
  }
}
```

### Inject project context on session start
```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "startup|resume",
      "hooks": [{
        "type": "command",
        "command": "cat .claude/context-reminder.txt"
      }]
    }]
  }
}
```

### Validate file writes (format check)
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "node scripts/validate-file-format.js"
      }]
    }]
  }
}
```

### Auto-allow specific permissions
```json
{
  "hooks": {
    "PermissionRequest": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "node scripts/auto-allow-safe-commands.js"
      }]
    }]
  }
}
```

## 8. Environment variables

| Variable | Available in |
|----------|-------------|
| `$CLAUDE_PROJECT_DIR` | All command hooks |
| `${CLAUDE_PLUGIN_ROOT}` | Plugin hooks |
| `${CLAUDE_PLUGIN_DATA}` | Plugin hooks |
| `$CLAUDE_CODE_REMOTE` | All (true when running remotely) |
| `$CLAUDE_ENV_FILE` | SessionStart only — append exports to persist env vars |

## 9. Key constraints

- **Stop hooks**: always check `stop_hook_active` field — if true, allow stopping to avoid infinite loop
- **Shell profiles**: echo statements in `.bashrc`/`.zshrc` break JSON parsing. Guard with `[[ $- == *i* ]]`
- **Timeouts**: command default 600s, http/prompt 30s, agent 60s
- **SessionEnd**: only 1.5s timeout — keep it fast
- **Async hooks**: fire-and-forget, no exit code processing, no output parsing
- **PostToolUse/Notification**: exit 2 shows error but does NOT block — use PreToolUse for blocking

## 10. Pre-send checklist

Before presenting a hook design:

1. [ ] Hook event matches the timing need (before vs after, blocking vs observing)
2. [ ] Matcher regex is correct and specific enough (not over-matching)
3. [ ] Handler type is the simplest that works (command > prompt > agent)
4. [ ] Exit codes are used correctly (0 = success, 2 = block for blocking events only)
5. [ ] Script handles missing/malformed stdin gracefully
6. [ ] Stop hooks check `stop_hook_active` to prevent infinite loops
7. [ ] Configuration location is appropriate (project vs user vs local)
8. [ ] Script is portable (no machine-specific paths unless user-level config)
9. [ ] Timeout is set appropriately for the task
10. [ ] Hook doesn't duplicate what a skill or CLAUDE.md rule already handles
