# Ralph Wiggum Loop + Beads

Autonomous AI agent loops powered by a local-first issue tracker and memory bank.

![Ralph Beads](ralph-beads.png)

## Beads

[Beads](https://github.com/steveyegge/beads) is a local-first issue tracker designed for AI agents. Issues live in `.beads/` as JSON files—no server, no sync conflicts, works offline. Hierarchical structure (epic → feature → task) with explicit dependencies. Issue descriptions double as memory: context persists across sessions without separate memory banks.

## Ralph Wiggum Loops

[Ralph Wiggum loops](https://github.com/ghuntley/how-to-ralph-wiggum) are autonomous execution patterns for AI coding agents. One bash loop. Read plan, pick task, implement, commit, exit. Fresh context every iteration—no accumulated cruft, no degraded attention. The AI operates in its "smart zone" continuously because the whiteboard wipes clean after each task.

## Combine Them!

Beads and Ralph Wiggum loops are effective standalone tools. Together, each strengthens the other.

Ralph usually tracks tasks in a flat file markdown doc with `passes` flags. It works. Beads improves it: hierarchical issues with explicit dependencies, atomic status updates (`bd close` vs editing markdown), and `bd ready` returns only unblocked work—no parsing task ordering from prose.

Beads tracks issues but needs an executor. Ralph provides it: pick issue, implement, close, exit. Fresh context each iteration.

The loop stays stateless; state lives in beads.

## What You Get

- **Atomic state** — Each agent claims and completes one issue at a time
- **Dependency ordering** — `bd ready` returns only unblocked work
- **Memory as issues** — Descriptions persist context across sessions
- **Hierarchical decomposition** — Epics → features → tasks with clear relationships

## How Beads Augments Ralph Wiggum Loops

| Traditional Ralph Approach | Problem | Beads Solution |
|---------------------|---------|----------------|
| `IMPLEMENTATION_PLAN.md` | Single file grows unbounded; crash loses edits; parallel workers conflict on writes | Atomic `bd close`; each worker claims own issue; no shared mutable file |
| Memory banks / `activity.md` | Separate system to maintain alongside tasks; context split across locations | Issue descriptions ARE the memory; context co-located with work |
| Flat task lists with `passes` flags | Agent must parse prose to determine ordering; no explicit dependencies | `bd ready` returns only unblocked work; dependencies are first-class |
| Full plan in context | Bloats context window; hallucination risk increases with prompt size | Each task is self-contained; related context is queryable, not preloaded |

## Beads Issue Hierarchy

```
epic → feature → task
bug (independent, any priority)
```

Supports up to 3 levels: `bd-a3f8e9` → `bd-a3f8e9.1` → `bd-a3f8e9.1.1`

**Priority**: P0 (critical) → P4 (backlog)

**Work order**: Priority first, then type (epic→feature→bug→task), then FIFO.

## How Ralph Handles Issues

For each ready issue, Ralph asks the model for a JSON plan:

| Field | Purpose |
|-------|---------|
| `has_enough_info` | Boolean. `false` means the issue is underspecified. |
| `missing` | Clarification gaps. Posted as a `bd` comment when `has_enough_info` is false, then the loop emits BLOCKED. |
| `implementation_steps` | What to do. |
| `verification_steps` | How to confirm it. |
| `closure_reason` | Passed to `bd close`. |

The loop validates the shape and executes mechanically. The scheduler does not dispatch on issue type — type, labels, and acceptance criteria are folded into the plan by the model. Epic/feature milestones work the same way: the plan names child-issue checks as verification steps, and the closure reason reflects whether all children are closed. See [prompt.md](prompt.md) for the full contract.

## PRD → Beads Workflow

1. **Write PRD**, then **prompt agent to create and decompose**:
   ```bash
   echo "Read [my-prd].md. Use bd to create an epic and decompose into tasks \
   with dependencies. Each task must have acceptance criteria." | claude --allowedTools "Bash(bd:*)"
   ```

2. **Agent does everything**:
   - Reads PRD
   - Creates epic with descriptive title, pointer in description
   - Creates child tasks with full requirements inline
   - Sets dependencies between tasks

3. **Verify and execute**:
   ```bash
   bd dep tree bd-a3f8e9    # Review structure
   ./ralph.sh               # Start execution
   ```

**Rules**:
- Epics point to PRDs; tasks are self-contained with all requirements inline
- All tasks must have acceptance criteria—agent cannot close a task until criteria pass

## How The Ralph+Beads Loop Works

The loop ([ralph.sh](ralph.sh)) is a thin wrapper—it just invokes Claude repeatedly and checks for completion signals. All the real logic lives in [prompt.md](prompt.md), which tells the agent how to select work, claim issues atomically, implement changes, verify, log completion, and commit.

Here's a high-level of what the [prompt.md](prompt.md) does when invoked with [ralph.sh](ralph.sh).
```
1. Check recent activity     → See what previous loops did
2. bd ready                  → Get next unblocked issue (if empty, stop)
3. Claim immediately         → bd update <id> --status=in_progress
4. Request JSON plan         → {has_enough_info, missing, implementation_steps, verification_steps, closure_reason}
5. Execute mechanically      → Implement → Verify → Log → Close → Commit → Push
6. Context clears, loop repeats
```

The loop is stateless. Context clears after each iteration. All state lives in beads.

## The Loop

See [`ralph.sh`](ralph.sh) for the full implementation.

```bash
./ralph.sh              # Run until all work complete
./ralph.sh --idle-sleep 30   # Custom sleep between checks
```

In a separate terminal window, you can watch raw output of ralph working:

```bash
tail -f logs/ralph-*.log     # Watch a stream of live, raw json output
```

## Cross-Loop Visibility

Traditional ralph uses `activity.md`—an append-only file for cross-loop memory. Beads replaces this with **ticket comments + `bd activity`**:

| Traditional | Beads Equivalent |
|-------------|------------------|
| Append to activity.md | `bd comments add <id> "..."` on the ticket |
| Read activity.md | `bd activity` + `bd comments <id>` |

**Ticket comments ARE the activity log.** Each loop writes a structured completion comment to the ticket:

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
```

### Rich Context (with comments)

`bd activity` shows state changes but not comment content. To see what was actually done in recent loops:

```bash
bd activity --limit 10 --json | jq -r '.[].issue_id' | sort -u | \
  xargs -I{} sh -c 'echo "=== {} ===" && bd comments {} 2>/dev/null'
```

### Recommended Claude Settings

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

## Origins

This approach synthesizes ideas from:
- [ghuntley/how-to-ralph-wiggum](https://github.com/ghuntley/how-to-ralph-wiggum) — The original Ralph pattern
- [ClaytonFarr/ralph-playbook](https://github.com/ClaytonFarr/ralph-playbook) — Structured playbook version
- [steveyegge/beads](https://github.com/steveyegge/beads) — Local-first issue tracker and memory bank for AI

## Dependencies

- `claude` CLI (Claude Code)
- `bd` CLI (beads)
- `git`
- `jq`

## License

MIT
