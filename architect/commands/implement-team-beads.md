---
description: "Execute beads using Agent Teams (experimental peer-to-peer coordination)"
argument-hint: "[--label <label>] [--epic <id>] [--workers <2-5>]"
allowed-tools:
  ["Bash", "Read", "TodoWrite", "AskUserQuestion"]
---

# Implement Team Beads Command

Execute beads using Agent Teams -- experimental peer-to-peer coordination where teammates self-claim tasks, communicate directly, and coordinate autonomously.

> **EXPERIMENTAL**: Agent Teams require `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in your environment or settings.json. Agent Teams have known limitations around session resumption, task coordination, and shutdown behavior.

## When to Use Agent Teams vs Other Modes

| Situation                                          | Recommended Mode   |
| -------------------------------------------------- | ------------------ |
| Independent beads, maximum speed                   | Swarm              |
| Sequential dependencies, review each step          | Loop               |
| Hands-off sequential                               | Auto Loop          |
| Complex cross-cutting changes, need peer communication | **Agent Teams** |
| Beads that may need to coordinate on shared files  | **Agent Teams**    |
| Debugging or investigating issues                  | Loop               |

**Key differences from Swarm/Loop:**

| Aspect          | Swarm (subagents)                  | Agent Teams                              |
| --------------- | ---------------------------------- | ---------------------------------------- |
| Communication   | Workers report to orchestrator only | Teammates message each other directly    |
| Task claiming   | Orchestrator assigns               | Teammates self-claim from shared list    |
| Dependencies    | Orchestrator manages               | Built-in, automatic unblocking           |
| Context         | Prompt context only                | Full Claude Code session with CLAUDE.md  |
| Token cost      | Lower                             | Higher (each teammate is a full session) |

## TDD Execution Flow

Same TDD pattern as all other modes:

1. **Test beads execute first** (create test files, must compile)
2. **Impl beads execute after** (implementation + test validation)
3. **Tests run after each impl bead** (must pass to complete)
4. **Blocked if tests fail**

TDD ordering is enforced through Agent Teams task dependencies.

## Arguments

- `--label <label>`: Filter beads by label (e.g., `--label openspec:my-change`) - skips epic selection
- `--epic <id>`: Execute beads for a specific epic - skips epic selection
- `--workers N`: Number of teammates to spawn (default: 3, min: 2, max: 5)

## Instructions

### Step 1: Validate Environment

```bash
# REQUIRED: Check Agent Teams enabled
if [ -z "$CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS" ] || [ "$CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS" != "1" ]; then
  echo "ERROR: Agent Teams not enabled"
  echo ""
  echo "Enable by adding to settings.json:"
  echo '  { "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }'
  echo ""
  echo "Or set in your shell:"
  echo "  export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1"
  exit 1
fi
echo "Agent Teams: enabled"

# Check bd installed
bd version

# Check initialized
bd ready &>/dev/null && echo "BD: OK" || echo "Run: bd init --stealth"

# Check Vitest configured
if ls vitest.config.ts vitest.config.js vitest.config.mts vitest.config.mjs 2>/dev/null | head -1; then
  echo "Vitest: OK"
else
  echo "WARNING: Vitest not configured - test validation will be skipped"
  echo "Install: npm install -D vitest"
fi
```

**If Agent Teams not enabled:** Stop immediately and show the error. Do NOT proceed.

### Step 2: Parse Arguments

Parse `$ARGUMENTS` for flags:

- `--label <value>` → skip epic selection, use this label
- `--epic <id>` → skip epic selection, use this epic
- `--workers <N>` (default: 3, clamped to 2-5)

### Step 3: Select Epic (Interactive Mode)

**Skip if** `--label` or `--epic` provided.

```bash
# Get all epics with openspec labels that are not completed
bd list -t epic --json 2>/dev/null | jq -r '
  .[] |
  select(.status == "pending" or .status == "in_progress") |
  "\(.id)\t\(.title)\t\(.status)\t\(.labels | map(select(startswith("openspec:"))) | join(","))"
'
```

**Present Available Epics:**

Use AskUserQuestion:

```
Use AskUserQuestion with:
- question: "Select an epic to implement with Agent Teams:"
- header: "Epic"
- options:
  - label: "<epic-title-1>"
    description: "<epic-id> | openspec:<name> | N ready beads"
  - label: "<epic-title-2>"
    description: "<epic-id> | openspec:<name> | N ready beads"
  ... (one option per available epic, max 4)
```

**Handle Selection:**

- If user selects an epic → extract the `openspec:<name>` label for filtering
- If user selects "Other" → use their custom input (label or epic ID)

**Edge Cases:**

- No epics found → "No epics found. Run `/create-beads` to create beads from an OpenSpec change."
- No ready beads → "No ready beads. All beads may be completed or blocked."

### Step 4: Retrieve Beads and Build Task Graph

```bash
bd list --json -l "<label>"
```

Parse the JSON to build the full dependency graph. For each bead, extract:

- `id`: Bead identifier
- `title`: Task name
- `depends_on`: Array of blocking bead IDs
- `status`: pending, in_progress, completed
- `labels`: Including `type:test`, `type:impl`, `no-test:*`, `openspec:*`

For each bead, get the full description:

```bash
bd show <id>
```

Also extract OpenSpec metadata:

1. Check for `openspec:*` label → extract change name (e.g., `openspec:add-auth` → `add-auth`)
2. Parse description for Context Chain → extract task number (e.g., `**Task**: 1.1 from tasks.md`)
3. If Context Chain is missing: set OpenSpec Task to `"unknown"`

**Calculate team size:**

```
ready_count = count of beads with status "pending" and all dependencies satisfied
team_size = min(--workers, ready_count, 5)
team_size = max(team_size, 2)
```

- If zero ready beads: "No ready beads. All beads may be completed or blocked." → EXIT

### Step 5: Create Agent Team

**CRITICAL: You are the LEAD. You coordinate ONLY. You do NOT implement beads yourself.**

Tell Claude to create an agent team with the following instruction:

---

Create an agent team with `<team_size>` teammates to implement code changes.

**Team name:** `beads-<label>` (e.g., `beads-openspec-add-auth`)

**Your role (lead):** You coordinate work. You do NOT write code, edit files, or run tests. You:

1. Create tasks from the bead list
2. Monitor teammate progress
3. Send messages if teammates need help or coordination
4. When a teammate reports a file conflict, mediate between the affected teammates
5. Synthesize results when all tasks are done
6. Clean up the team

**DO NOT enable delegate mode (Shift+Tab).** Stay in normal mode but restrict yourself to coordination only.

**Teammate names:** `worker-1`, `worker-2`, ... `worker-N`

**Teammate spawn prompt** (same for all teammates):

```
You are a bead implementation worker. You claim tasks from the shared task list, implement them, and mark them complete.

## Your Workflow

REPEAT until no tasks remain:
  1. Check the task list for available (pending, unblocked) tasks
  2. Claim a task
  3. Read the full task description carefully - it contains everything you need
  4. Implement the changes described
  5. Run verification/exit criteria commands from the task
  6. For impl beads: run `npx vitest run <test-file> --reporter=verbose`
  7. If successful: update bead status and OpenSpec, mark task complete
  8. If blocked: report failure, do NOT mark task complete, message the lead
  9. Claim next task

## After Completing a Bead

For EVERY completed bead, you MUST do ALL of these steps:

### A. Update Bead Status (REQUIRED)
```bash
bd close <bead-id> --reason "<brief summary of what was done>"
```

### B. Update OpenSpec tasks.md (if applicable)
If the task mentions an OpenSpec label and task number:
1. Open: `openspec/changes/<change-name>/tasks.md`
2. Find: `- [ ] <task-number>` (e.g., `- [ ] 1.1`)
3. Change: `[ ]` to `[x]`

Example:
- Before: `- [ ] 1.1 Create useResponsive hook`
- After:  `- [x] 1.1 Create useResponsive hook`

If the task number is "unknown", skip this step.

### C. If Blocked
```bash
bd update <bead-id> --status blocked
```
Then message the lead explaining why you are blocked.

## Architecture Guide

BEFORE writing ANY implementation code, you MUST:
1. Check if `docs/architecture/INDEX.md` exists
2. If it does: read it, follow the Quick Reference table, read only the relevant architecture docs for your task
3. If it does not exist: proceed without architecture docs

## Test Validation (impl beads only)

For beads with label `type:impl`:
1. Find the test file path from the task description
2. Run: `npx vitest run <test-file-path> --reporter=verbose`
3. If tests PASS → mark bead complete
4. If tests FAIL → mark bead BLOCKED with error summary

## Rules

- NEVER push to git (no `git push`, no `bd sync`)
- NEVER skip tests for impl beads
- If you encounter merge conflicts with another teammate, message them directly to coordinate
- If you finish all available tasks, go idle
- Match existing code style in the codebase
```

---

### Step 6: Create Tasks from Beads

After the team is created, create one Agent Teams task per bead in the shared task list.

**Task creation order (enforces TDD):**

1. First: all test beads (`type:test`) with no bead dependencies
2. Then: test beads that depend on other test beads (set task dependency)
3. Then: impl beads (`type:impl`) — each depends on its corresponding test bead task
4. Then: non-testable beads (`no-test:*`)

**For each bead, create a task with:**

- **Subject:** `[<bead-id>] <bead-title>`
- **Description:**

```
Bead ID: <id>
Bead Type: <test | impl | non-testable>
OpenSpec Label: <openspec:change-name or "none">
OpenSpec Task: <task number or "unknown">

## Full Bead Description
<complete output of `bd show <id>`>

## Verification Commands
<exit criteria from bead description>

## Test Validation (impl beads only)
Test File: <path>
Command: npx vitest run <test-file-path> --reporter=verbose
```

**Task dependencies:** If bead B depends on bead A, set B's task to depend on A's task. The Agent Teams system will automatically prevent teammates from claiming B until A is complete.

**Mark all beads as in_progress:**

```bash
bd update <id> --status in_progress
```

Run this for ALL beads being added as tasks.

### Step 7: Monitor Progress

The lead monitors the team through the shared task list and teammate messages.

**Monitoring behavior:**

- If a teammate messages about being BLOCKED: log the issue, check if it's a test failure or dependency problem, message other teammates if coordination is needed
- If a teammate goes idle with tasks remaining: message them "There are still pending tasks. Please check the task list and claim the next available task."
- If two teammates report conflicting file edits: message both to coordinate which one takes priority
- If a teammate crashes or stops unexpectedly: spawn a replacement teammate with the same prompt
- **Do NOT implement anything yourself. Only coordinate.**

**Progress display (show periodically):**

```
===============================================================
AGENT TEAM PROGRESS
===============================================================

Team: beads-<label>
Teammates: <N> active

Tasks:
  Completed: X / Total
  In Progress: Y
  Pending: Z
  Blocked: W

Active Teammates:
  worker-1: [<bead-id>] <title> (in_progress)
  worker-2: [<bead-id>] <title> (in_progress)
  worker-3: idle

===============================================================
```

### Step 8: Finalize and Clean Up

When all tasks are complete or blocked:

**1. Verify bead statuses match task statuses:**

```bash
bd list -l "<label>" --json
```

For any task marked complete in the team but NOT closed in bd:

```bash
bd close <id> --reason "Completed by agent team"
```

**2. Shut down all teammates:**

Ask each teammate to shut down gracefully. Wait for confirmations.

**3. Clean up the team:**

```
Clean up the team
```

**4. Report results:**

```
===============================================================
IMPLEMENTATION COMPLETE (AGENT TEAMS)
===============================================================

Team: beads-<label>
Teammates: <N>
Mode: Agent Teams (experimental)

Results:
  Test Beads: X completed (test files created)
  Impl Beads: Y completed (tests passing)
  Non-Testable: Z completed
  Blocked: W beads
  Remaining: R beads

Test Summary:
  Total Test Files: N
  Tests Passing: P
  Tests Failing: F

===============================================================
```

## Error Handling

| Scenario                       | Action                                                                                  |
| ------------------------------ | --------------------------------------------------------------------------------------- |
| Agent Teams not enabled        | "Set CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 in settings.json or environment"            |
| No epics found                 | "No epics found. Run `/create-beads` to create beads from an OpenSpec change."          |
| No ready beads                 | "No ready beads. All beads may be completed or blocked."                                |
| Vitest not configured          | "WARNING: Test validation will be skipped. Install: `npm install -D vitest`"            |
| Tests fail (impl bead)         | Teammate marks bead as blocked, messages lead                                           |
| Teammate crashes               | Lead spawns replacement teammate, reassigns unclaimed tasks                             |
| Teammate goes idle prematurely | Lead messages teammate to check task list                                               |
| File conflict between teammates | Lead mediates, tells one teammate to wait for the other                                |
| All beads blocked              | Stop, report blocking issues, clean up team                                             |

## Git Policy

**NEVER push to git.** Do not run `git push`, `bd sync`, or any command that pushes to remote. The user will push manually when ready.

## Optional: Quality Gate Hooks

For additional safety, configure a `TaskCompleted` hook to verify bead completion:

```json
{
  "hooks": {
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'BEAD_ID=$(echo \"$CLAUDE_TASK_SUBJECT\" | grep -oP \"\\[\\K[^\\]]+\") && TYPE=$(bd show $BEAD_ID 2>/dev/null | grep -o \"type:impl\" | head -1) && if [ \"$TYPE\" = \"type:impl\" ]; then TEST_FILE=$(bd show $BEAD_ID | grep -oP \"Test File: \\K.*\" | head -1) && npx vitest run \"$TEST_FILE\" --reporter=verbose; fi'"
          }
        ]
      }
    ]
  }
}
```

This hook extracts the bead ID from the task subject, checks if it's an impl bead, and runs vitest. If tests fail, it exits with code 2 to block task completion.

## Context Recovery

If you lose track of progress:

```bash
bd ready                        # See what's next
bd list --status in_progress    # Find current work
bd list -l "<label>"            # See all beads for this label
bd show <id>                    # Full task details
```

## Stopping

- Run `/cancel-implementation` to stop early
- Tell the lead: "Shut down all teammates and clean up the team"
- Teammates may finish their current task before stopping

## Example Usage

```bash
# Interactive mode (select epic)
/implement-team-beads

# With specific label
/implement-team-beads --label openspec:add-auth

# With 5 teammates
/implement-team-beads --label openspec:add-auth --workers 5

# Minimal team (2 teammates)
/implement-team-beads --label openspec:refactor --workers 2
```
