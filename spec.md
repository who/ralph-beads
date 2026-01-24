# Ralph Wiggum + Beads

**Beads** is a local-first issue tracker for AI agents.
1. **Issue tracker** → Hierarchical issues with dependencies
2. **Memory bank** → Descriptions persist context across sessions
3. **Exploded PRD** → Epics point to PRD docs; tasks contain full requirements inline

**Ralph loops** are autonomous agent execution cycles that consume work from a queue.
1. **Stateless** → Context clears after each iteration; state lives in beads + git
2. **Mode-switched** → Issue type determines behavior (planning vs building)
3. **Self-terminating** → Loop exits when `bd ready` returns empty

## Origins

- [ghuntley/how-to-ralph-wiggum](https://github.com/ghuntley/how-to-ralph-wiggum)
- [ClaytonFarr/ralph-playbook](https://github.com/ClaytonFarr/ralph-playbook)
- [steveyegge/beads](https://github.com/steveyegge/beads)

## The Pattern

**Discovery (if requirements are unclear):**
```bash
echo "I want to build [brief product idea]. Use AskUserQuestion to clarify requirements \
through 3-5 rounds of questions. Think deeply about architecture, edge cases, and \
dependencies. Then write a PRD to prd/[feature].md and decompose it into beads tasks \
with dependencies. Each task must have acceptance criteria." | claude
```

**Setup (if PRD already exists):**
```bash
echo "Read prd/auth-system.md. Use bd to create an epic and decompose into tasks with \
dependencies. Each task must have self-contained requirements and acceptance criteria." | claude
```

**Loop (repeats until done):**
```
1. bd ready              → Get next unblocked issue
2. Issue type decides mode:
   - epic/feature        → Planning (decompose into child issues)
   - task/bug            → Building (implement code)
3. bd close <id>         → Complete, context clears
4. Loop repeats
```

## Why Beads

| Traditional Approach | Problem | Beads Solution |
|---------------------|---------|----------------|
| `IMPLEMENTATION_PLAN.md` | Crash loses edits; workers conflict | Atomic `bd close`; each claims own issue |
| Memory banks / `.claude/` | Separate system to maintain | Issue descriptions ARE the memory |
| `prd/*.md` files | Disconnected from execution | Epics/features ARE requirements |
| Flat task lists | Agent must parse ordering | `bd ready` returns only unblocked work |
| Full plan in context | Hallucination risk | Task sees parent only |

## Issue Type Rules

All rules are in **prompts/prompt.md**:

| Type | Behavior |
|------|----------|
| **task** | Implement exactly what's specified, no scope expansion |
| **bug** | Fix only the bug, add regression test, no unrelated changes |
| **epic/feature** | Milestone check—close when all children complete |

## File Structure

```
project-root/
├── ralph.sh             # Orchestration loop
├── AGENTS.md            # Operational constraints
├── prompts/
│   └── prompt.md        # All instructions (includes issue type rules)
├── prd/                 # Large requirement docs (optional)
└── .beads/              # Issues point to PRDs, track state
```

## The Loop

```bash
#!/bin/bash
# ralph.sh - Autonomous task execution loop
#
# Usage: ./ralph.sh [--idle-sleep N]
#
# Runs until all ready work is complete. Logs to logs/ralph-<timestamp>.log
# Watch live: tail -f logs/ralph-*.log

set -e

IDLE_SLEEP=60

while [[ $# -gt 0 ]]; do
  case $1 in
    --idle-sleep) IDLE_SLEEP="$2"; shift 2 ;;
    -h|--help) head -n 8 "$0" | tail -n +2 | sed 's/^# //'; exit 0 ;;
    *) echo "Unknown option: $1" >&2; exit 1 ;;
  esac
done

mkdir -p logs
LOG_FILE="logs/ralph-$(date '+%Y%m%d-%H%M%S').log"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

log "=== Ralph Started ==="
log "Log file: $LOG_FILE"
log "Watch live: tail -f $LOG_FILE"

tasks_completed=0

while true; do
  log ""
  log "--- Checking for ready issues ---"

  ISSUE=$(bd ready --json 2>/dev/null | jq -r '.[0] // empty')

  if [ -n "$ISSUE" ]; then
    ID=$(echo "$ISSUE" | jq -r '.id')
    TYPE=$(echo "$ISSUE" | jq -r '.type')
    TITLE=$(echo "$ISSUE" | jq -r '.title')

    log "Found: $ID ($TYPE) - $TITLE"
    log ""
    log "========================================"
    log "Starting: $ID"
    log "========================================"

    bd update "$ID" --status=in_progress

    result=$(claude -p "$(cat prompts/prompt.md)" --output-format stream-json --verbose --dangerously-skip-permissions 2>&1 | tee -a "$LOG_FILE") || true

    if [[ "$result" == *"<promise>COMPLETE</promise>"* ]] || [[ "$result" == *"COMPLETE"* ]]; then
      tasks_completed=$((tasks_completed + 1))
      log "Task $ID completed. Total: $tasks_completed"
    elif [[ "$result" == *"<promise>BLOCKED</promise>"* ]] || [[ "$result" == *"BLOCKED"* ]]; then
      log "Task $ID blocked. Check beads comments for details."
    fi
  else
    if [ "$tasks_completed" -gt 0 ]; then
      log ""
      log "========================================"
      log "No more ready work. Tasks completed: $tasks_completed"
      log "========================================"
      exit 0
    else
      log "No work found. Sleeping ${IDLE_SLEEP}s... (Ctrl+C to stop)"
      sleep "$IDLE_SLEEP"
    fi
  fi
done
```

## Issue Hierarchy

```
epic → feature → task
bug (independent, any priority)
```

Supports up to 3 levels: `bd-a3f8e9` → `bd-a3f8e9.1` → `bd-a3f8e9.1.1`

**Priority**: P0 (critical) → P4 (backlog)

**Work order**: Priority first, then type (epic→feature→bug→task), then FIFO.

## PRD → Beads Workflow

1. **Write PRD**, then **prompt agent to create and decompose**:
   ```bash
   echo "Read prd/auth-system.md. Use bd to create an epic (title from PRD, description points to file) and decompose into tasks with dependencies. Each task must have self-contained requirements and acceptance criteria." | claude
   ```

2. **Agent does everything**:
   - Reads PRD
   - Creates epic with descriptive title, pointer in description
   - Creates child tasks with full requirements inline
   - Sets dependencies between tasks

3. **Verify and execute**:
   ```bash
   bd dep tree bd-a3f8e9    # Review structure
   ./loop.sh                 # Start execution
   ```

**Rules**:
- Epics point to PRDs; tasks are self-contained with all requirements inline
- All tasks must have acceptance criteria—agent cannot close a task until criteria pass

## Key Principles

1. One issue per iteration—context clears after each
2. Planning agents don't write code
3. Building agents don't expand scope
4. Parallel subagents for reads; single subagent for build/test
5. Don't assume not implemented—search first

## Cross-Loop Visibility

Traditional ralph uses `activity.md`—an append-only file for cross-loop memory. Beads replaces this with **ticket comments + `bd activity`**:

| Traditional | Beads Equivalent |
|-------------|------------------|
| Append to activity.md | `bd comments add <id> "..."` on the ticket |
| Read activity.md | `bd activity` + `bd comments <id>` |

**Ticket comments ARE the activity log.** Each loop writes a structured completion comment to the ticket (matching the ortus activity.md format):

```
**Changes**:
- <file or component> - <what was done>
- <another change>

**Verification**: <test results, lint status, manual checks>
```

This keeps activity co-located with the work rather than in a separate file.

### `bd activity` (state changes)

```bash
bd activity                     # Show last 100 events
bd activity --since 5m          # Events from last 5 minutes
bd activity --follow            # Real-time streaming
bd activity --mol bd-x7k        # Filter by issue prefix
bd activity --type update       # Filter by event type
bd activity --town              # Aggregated feed from all rigs
```

**Event symbols**:
| Symbol | Meaning |
|--------|---------|
| `+` | created/bonded |
| `→` | in_progress |
| `✓` | completed |
| `✗` | failed |
| `⊘` | deleted |

**Loop integration**: AI runs `bd activity --limit 10` at loop start to see recent state changes across all issues.

### Rich Context (with comments)

`bd activity` shows state changes but not comment content. To see what was actually done in recent loops:

```bash
# Get recent activity with full comment content
bd activity --limit 10 --json | \
  jq -r '.[].issue_id' | \
  sort -u | \
  while read -r id; do
    echo "=== $id ==="
    bd comments "$id" 2>/dev/null || echo "(no comments)"
    echo
  done
```

Or as a one-liner for prompt.md:

```bash
bd activity --limit 10 --json | jq -r '.[].issue_id' | sort -u | xargs -I{} sh -c 'echo "=== {} ===" && bd comments {} 2>/dev/null'
```

## Dependencies

- `claude` CLI
- `bd` CLI (beads)
- `git`
- `jq`

## Claude Integration

Beads uses a **CLI + Hooks** approach for Claude Code integration—lightweight, editor-agnostic, and context-efficient.

### `bd prime`

Injects ~1-2k tokens of workflow context into Claude's session:

```bash
bd prime              # Output workflow context to stdout
bd prime --check      # Verify prime content is valid
```

This teaches Claude the beads workflow (ready → create → update → close → sync) without bloating the context window.

### Hooks

Two hooks automate context injection:

| Hook | Trigger | Purpose |
|------|---------|---------|
| **SessionStart** | Claude Code session begins | Runs `bd prime` to inject workflow context |
| **PreCompact** | Before context compaction | Preserves beads instructions when Claude summarizes long conversations |

### Setup

```bash
bd setup claude             # Global installation
bd setup claude --project   # Project-only (writes to .claude/)
bd setup claude --stealth   # Flush only, no git operations
bd setup claude --check     # Verify installation
bd setup claude --remove    # Remove hooks
```

### Verification

```bash
bd doctor claude            # Check integration health
```

### Why Hooks Over MCP/Skills

| Approach | Context Cost | Trade-off |
|----------|--------------|-----------|
| MCP tool schemas | 10-50k tokens | Full tool introspection, but heavy |
| Claude Skills | Variable | Claude-specific, breaks other editors |
| **`bd prime` + hooks** | **1-2k tokens** | 10-50x lighter, works everywhere |

**Design priorities**:
- **Context efficiency** — Attention quality degrades with prompt size
- **Editor-agnostic** — Same workflow for Claude Code, Cursor, Windsurf, Zed
- **Simplicity** — Beads workflow is 5 commands, not a plugin ecosystem

### Claude Settings

`.claude/settings.json` for ralph loops:

```json
{
  "permissions": {
    "allow": [
      "WebFetch(domain:github.com)",
      "WebFetch(domain:docs.anthropic.com)",
      "Bash(bd *)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git add:*)",
      "Bash(git commit:*)"
    ],
    "deny": [
      "Bash(sudo *)",
      "Bash(rm -rf *)",
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Read(**/credentials.json)"
    ],
    "ask": [
      "Bash(git push:*)"
    ],
    "defaultMode": "acceptEdits"
  },
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": false,
    "network": {
      "allowLocalBinding": true
    }
  }
}
```
