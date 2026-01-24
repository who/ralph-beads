# Ralph Beads

Autonomous AI agent loops powered by a local-first issue tracker.

## The Problem

AI coding agents lose context. They hallucinate plans. They forget what they were doing. Multiple agents conflict. Session crashes lose work.

Existing solutions create more problems:
- `IMPLEMENTATION_PLAN.md` files get corrupted by concurrent edits
- Memory banks require a separate system to maintain
- PRD documents sit disconnected from actual execution
- Flat task lists force agents to parse ordering themselves

## The Solution

**Beads** is a local-first issue tracker designed for AI agents. **Ralph loops** are autonomous execution cycles that consume work from beads.

Together they provide:
- **Atomic state** — Each agent claims and completes one issue at a time
- **Dependency ordering** — `bd ready` returns only unblocked work
- **Memory as issues** — Descriptions persist context across sessions
- **Hierarchical decomposition** — Epics → features → tasks with clear relationships

## How It Works

```
1. Agent runs `bd ready` to get the next unblocked issue
2. Issue type determines behavior:
   - task/bug → Implement code
   - epic/feature → Check if children complete (milestone)
3. Agent completes work, runs `bd close <id>`
4. Loop repeats until no ready work remains
```

The loop is stateless. Context clears after each iteration. All state lives in beads + git.

## Cross-Loop Visibility

Traditional ralph uses `activity.md` for cross-loop memory. Beads replaces this with **ticket comments + `bd activity`**:

```bash
# At loop start: see recent activity with comment details
bd activity --limit 10 --json | jq -r '.[].issue_id' | sort -u | \
  xargs -I{} sh -c 'echo "=== {} ===" && bd comments {} 2>/dev/null'
```

Each loop writes a structured completion comment to the ticket:

```
**Changes**:
- Added auth middleware in src/middleware/auth.ts
- Created login/logout endpoints

**Verification**: All tests passing, lint clean
```

Activity is co-located with work—no separate file to maintain.

## Quick Start

```bash
# Discovery: unclear requirements
echo "I want to build [product idea]. Use AskUserQuestion to clarify requirements \
through 3-5 rounds of questions. Write a PRD to prd/[feature].md and decompose \
into beads tasks with dependencies." | claude

# Setup: PRD already exists
echo "Read prd/auth-system.md. Use bd to create an epic and decompose into tasks \
with dependencies. Each task must have acceptance criteria." | claude

# Execute: run the loop
./ralph.sh
```

## Origins

This approach synthesizes ideas from:
- [ghuntley/how-to-ralph-wiggum](https://github.com/ghuntley/how-to-ralph-wiggum) — The original Ralph pattern
- [ClaytonFarr/ralph-playbook](https://github.com/ClaytonFarr/ralph-playbook) — Structured playbook version
- [steveyegge/beads](https://github.com/steveyegge/beads) — Local-first issue tracker for AI

## Dependencies

- `claude` CLI (Claude Code)
- `bd` CLI (beads)
- `git`
- `jq`

## Claude Integration

Beads uses CLI + Hooks for Claude Code integration (~1-2k tokens vs 10-50k for MCP):

```bash
bd setup claude        # Install hooks
bd doctor claude       # Verify integration
```

Hooks run `bd prime` at session start and before context compaction to inject workflow context.

## Files

```
project-root/
├── ralph.sh             # Orchestration loop
├── AGENTS.md            # Operational constraints
├── prompts/
│   └── prompt.md        # Agent instructions
├── prd/                 # PRD documents (optional)
└── .beads/              # Issue state
```

## License

MIT
