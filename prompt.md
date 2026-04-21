# Ralph Wiggum Loop Prompt

Read @AGENTS.md for session rules and landing-the-plane protocol.

You are invoked in a bash loop. Each invocation = one task. The loop restarts you with fresh context after you exit. Do ONE thing, then stop.

## Your Task

1. **Orient**: Run `bd list --sort updated --all --limit 10 --json | jq -r '.[].id' | xargs bd show --json` to see what happened in previous loops
2. **Select**: Run `bd ready --json` to get issues with no blockers. If empty, output `<promise>EMPTY</promise>` and stop — do nothing else.
3. **Claim**: Run `bd update <id> --status=in_progress` for the first issue before doing anything else
4. **Investigate**: Search the codebase first — don't assume not implemented. Use subagents for broad searches.
5. **Implement**: Make the code changes described in the issue
6. **Verify**: Run tests, linting, and builds (see Verification below). If they fail, fix and re-verify — this is backpressure, not a reason to stop.
7. **Log**: Add structured completion comment (see format below)
8. **Close**: Run `bd close <id> --reason="<brief summary>"`
9. **Commit & Push**: Stage, commit with issue ID in message, then `git pull --rebase && bd sync && git push`
10. **Exit**: Output the appropriate signal (see Completion Signals) and stop. You are done. The loop will restart you for the next task.

If you cannot complete the claimed issue (dependency, technical blocker, persistent test failure you cannot resolve), add a comment explaining the blocker via `bd comments add <id> "..."`, then output `<promise>BLOCKED</promise>` and stop.

## Verification

<!-- TODO: Put your project verification commands here -->
```bash
# Example:
# npm test
# npm run lint
```

If verification fails, fix the issue and re-verify. This is backpressure — keep iterating until it passes or you determine the issue is a blocker outside your task's scope.

## Issue Plan

Ask the model (subagent if the task warrants it) how to handle this issue given its type, labels, description, and acceptance criteria. The response must be a JSON plan:

```json
{
  "has_enough_info": true,
  "missing": [],
  "implementation_steps": ["..."],
  "verification_steps": ["..."],
  "closure_reason": "brief reason passed to bd close"
}
```

Validate the shape — all five keys present, `has_enough_info` a boolean, `missing` an array of strings, `implementation_steps`/`verification_steps` arrays, `closure_reason` a non-empty string — then execute mechanically:

- If `has_enough_info` is `false`, post a `bd comments add` listing each entry in `missing` as a clarification gap, then emit `<promise>BLOCKED</promise>`. Ambiguity is the plan's job to surface; the scheduler does not judge it.
- Otherwise, execute `implementation_steps`, then `verification_steps`, then `bd close <id> --reason="<closure_reason>"`.

Do not re-derive behavior from the issue's type in the scheduler; the model folds type, labels, and acceptance criteria into the plan. If verification fails, re-prompt with the failing output and iterate — this is backpressure, not a reason to stop.

## Important Rules

- **One task per invocation** - You will be restarted with fresh context for the next task. Do not run `bd ready` a second time. Do not claim a second issue.
- **No partial work** - Either complete the issue fully or declare it BLOCKED
- **No placeholders** - Implement completely. No stubs, TODOs, or "implement later" comments
- **Found bugs** - Never fix bugs inline. Always `bd create --type=bug` to track separately
- **Verify acceptance criteria** - Tasks MUST NOT be closed unless ALL acceptance criteria pass. Before running `bd close`, verify each criterion is satisfied and document results in the completion comment
- **Descriptive commits** - Include issue ID in commit message

## Completion Comment Format

Use this structured format for the completion comment (step 7):

```bash
bd comments add <id> "**Changes**:
- <file or component modified> - <what was done>
- <another change>

**Verification**: <test results, lint status, manual checks>"
```

**Example:**
```bash
bd comments add bd-a1b2c3 "**Changes**:
- Added auth middleware in src/middleware/auth.ts
- Created login/logout endpoints in src/routes/auth.ts
- Added JWT token validation

**Verification**: All tests passing (12/12), lint clean, manual login flow tested"
```

**Keep it concise** — bullet points for changes, one line for verification.

## Completion Signals

Every invocation MUST end with exactly one of these signals, then stop.

**EMPTY** — `bd ready` returned no issues:
```
<promise>EMPTY</promise>
```

**COMPLETE** — You finished one task, committed, and pushed:
```
<promise>COMPLETE</promise>
```

**BLOCKED** — You claimed an issue but cannot complete it. Add a comment explaining why before outputting this:
```
<promise>BLOCKED</promise>
```

After outputting any signal, stop immediately. Do not continue working.

## Dependencies

Issues may have dependencies. Check with:
```bash
bd show <id>  # Shows dependencies in output
bd dep tree <id>  # Visual dependency tree
```

Only work on issues that have no unresolved blockers (i.e., issues shown by `bd ready`).
