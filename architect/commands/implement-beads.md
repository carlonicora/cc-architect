---
description: "Execute beads using loop (sequential) or swarm (parallel) mode"
argument-hint: "[--label <label>] [--epic <id>] [--mode <swarm|loop|auto>]"
allowed-tools:
  ["Task", "TaskOutput", "Read", "TodoWrite", "Bash", "Edit", "AskUserQuestion"]
---

# Implement Command

Execute beads using your choice of execution strategy.

**IMPORTANT**: This command combines loop (sequential) and swarm (parallel) execution modes with interactive selection.

## Execution Modes

| Mode          | Description                               | Best For                         |
| ------------- | ----------------------------------------- | -------------------------------- |
| **Swarm**     | Parallel workers                          | Independent beads, maximum speed |
| **Loop**      | Sequential with step-by-step confirmation | Interdependent tasks, debugging  |
| **Auto Loop** | Sequential without confirmation           | Hands-off sequential execution   |

## TDD Execution Flow

All modes follow the same TDD pattern:

1. **Test beads execute first** (create test files, must compile)
2. **Impl beads execute after** (implementation + test validation)
3. **Tests run after each impl bead** (must pass to complete)
4. **Blocked if tests fail** (in all modes)

## Arguments

- `--label <label>`: Filter beads by label (e.g., `--label openspec:my-change`) - skips epic selection
- `--epic <id>`: Execute beads for a specific epic - skips epic selection
- `--mode <swarm|loop|auto>`: Skip mode selection prompt
- `--workers N`: Maximum concurrent workers for swarm mode (default: 5)
- `--model MODEL`: Worker model for swarm - `haiku`, `sonnet`, or `opus` (default: opus)

## Instructions

### Step 1: Validate Environment

```bash
# Check bd installed
bd version

# Check initialized
bd ready &>/dev/null && echo "OK" || echo "Run: bd init --stealth"

# Check Vitest configured (REQUIRED for test validation)
if ls vitest.config.ts vitest.config.js vitest.config.mts vitest.config.mjs 2>/dev/null | head -1; then
  echo "Vitest OK"
else
  echo "WARNING: Vitest not configured - test validation will be skipped"
  echo "Install: npm install -D vitest"
fi
```

### Step 2: Parse Arguments

Parse `$ARGUMENTS` for flags:

- `--label <value>` → skip epic selection, use this label
- `--epic <id>` → skip epic selection, use this epic
- `--mode <swarm|loop|auto>` → skip mode selection
- `--workers <N>` (default: 5)
- `--model <MODEL>` (default: opus)

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
- question: "Select an epic to implement:"
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

### Step 4: Select Execution Mode (Interactive Mode)

**Skip if** `--mode` provided.

Use AskUserQuestion:

```
Use AskUserQuestion with:
- question: "Select execution mode:"
- header: "Mode"
- options:
  - label: "Swarm (Recommended)"
    description: "Parallel workers - maximum speed, up to N concurrent tasks"
  - label: "Loop"
    description: "Sequential - pause after each bead for review"
  - label: "Auto Loop"
    description: "Sequential - no pauses, runs until complete"
```

### Step 5: Execute Based on Mode

---

## MODE: SWARM

Execute beads using parallel worker agents.

### Swarm Step 1: Retrieve Beads

```bash
bd list --json -l "<label>"
```

Parse the JSON to build a task graph with dependencies.

### Swarm Step 2: Build Dependency Graph

From the beads JSON, extract:

- `id`: Bead identifier
- `title` or `description`: Task name
- `depends_on`: Array of blocking bead IDs
- `status`: pending, in_progress, completed

**A bead is "ready" when:**

- Status is `pending`
- All beads in `depends_on` have status `completed`

### Swarm Step 2.5: Extract OpenSpec Metadata

For each bead, before spawning a worker, extract OpenSpec metadata:

1. Check if bead has an `openspec:*` label
2. If yes, extract the change name (e.g., `openspec:add-auth` → `add-auth`)
3. Get the full bead description with `bd show <id>`
4. Parse the description to find the task number:
   - Look for `**Task**: X.X from tasks.md` in the Context Chain section
   - Extract the task number (e.g., "1.1", "2.3")

**If Context Chain is missing or malformed:**

- Set OpenSpec Task to `"unknown"`
- Worker will skip OpenSpec update and log a warning
- The bead can still complete successfully (implementation is valid)

### Swarm Step 3: Spawn Workers

For each ready bead (up to `--workers` limit):

1. Mark bead as in_progress:

   ```bash
   bd update <id> --status in_progress
   ```

2. Launch worker agent using Task tool:

   ```
   Use Task tool with:
   - subagent_type: "general-purpose"
   - model: <selected model>
   - run_in_background: true
   - prompt: |
       Execute this bead task:

       Bead ID: <id>
       Title: <title>
       Bead Type: <test | impl | non-testable>
       OpenSpec Label: <openspec:change-name or "none">
       OpenSpec Task: <task number from Context Chain, e.g., "1.1", or "unknown" if not found>

       ## Task Description
       <full bead description from bd show>

       ## Instructions
       1. Read the files mentioned
       2. Make the required changes (use LSP tools for TypeScript: documentSymbol, findReferences, goToDefinition)
       3. Run verification commands in exit criteria

       ## Test Validation (for impl beads only)
       If this bead has label `type:impl`:
       4. Find associated test file from bead description
       5. Run tests: `npx vitest run <test-file-path> --reporter=verbose`
       6. If tests FAIL: Output "BEAD BLOCKED: <id> - Tests failed: <summary>"
       7. If tests PASS: Continue to completion

       ## OpenSpec Update (REQUIRED if OpenSpec Label is not "none")

       8. Mark the task as complete in tasks.md:
          - If OpenSpec Task is "unknown", skip this step and log: "WARNING: Could not find task number in Context Chain, skipping OpenSpec update"
          - Open: `openspec/changes/<change-name>/tasks.md` (use change-name from OpenSpec Label)
          - Find the line starting with: `- [ ] <OpenSpec Task>` (e.g., `- [ ] 1.1`)
          - Change `[ ]` to `[x]` (keep everything else the same)

          Example:
          - Before: `- [ ] 1.1 Create useResponsive hook`
          - After:  `- [x] 1.1 Create useResponsive hook`

       9. Report success or failure

       When complete, output: "BEAD COMPLETE: <id>"
       If blocked, output: "BEAD BLOCKED: <id> - <reason>"
   ```

3. Track worker assignment: `worker_id → bead_id`

### Swarm Step 4: Monitor and Manage Workers (CRITICAL LOOP)

**CRITICAL: You MUST stay in this monitoring loop until ALL beads are complete or blocked. DO NOT exit after spawning workers.**

```
WHILE (pending_beads > 0 OR active_workers > 0):

    1. WAIT for any worker to complete:
       - Call TaskOutput(task_id, block=true, timeout=60000) for each active worker
       - This blocks until a worker finishes or times out

    2. WHEN a worker completes:
       - Parse output for "BEAD COMPLETE: <id>" or "BEAD BLOCKED: <id>"
       - If COMPLETE:
         - Run: bd close <id> --reason "Done"
         - Remove from active_workers
       - If BLOCKED:
         - Run: bd update <id> --status blocked
         - Report the blocking reason
         - Remove from active_workers

    3. UPDATE dependency graph:
       - Refresh ready beads: bd ready -l "<label>" --json
       - Identify newly unblocked beads (dependencies now satisfied)

    4. SPAWN replacement workers:
       - For each newly ready bead (up to --workers limit):
         - Mark as in_progress
         - Launch worker agent (same as Swarm Step 3)
         - Add to active_workers

    5. REPORT progress:
       - Show: "Progress: X/Y beads complete, Z active workers, W blocked"

    6. CONTINUE the loop (go back to step 1)

END WHILE
```

**Exit conditions (ONLY exit when one of these is true):**

- All beads have status `completed` → Success
- All remaining beads are `blocked` with no active workers → Blocked state
- No more ready beads AND no active workers → Done

**DO NOT exit just because you spawned workers. You MUST wait for them to finish and spawn new workers for unblocked beads.**

---

## MODE: LOOP / AUTO LOOP

Execute beads iteratively until all ready tasks are complete.

### Loop Step 1: Find Ready Work

```bash
bd ready -l "<label>"
```

Shows tasks with no blockers, sorted by priority.

### Loop Step 2: Pick a Task

Select the highest priority ready task. Note its ID.

### Loop Step 3: Read Task Details

```bash
bd show <id>
```

The task description should be self-contained with requirements, acceptance criteria, and files to modify.

### Loop Step 4: Start Working

```bash
bd update <id> --status in_progress
```

### Loop Step 5: Implement the Task

Follow the task description:

1. Read the files mentioned
2. Make the required changes
3. Run any tests/verification in acceptance criteria

**TypeScript LSP (when available):** For TypeScript/JavaScript code, use LSP tools for better accuracy:

- `documentSymbol` - Find symbols in a file
- `findReferences` - Find all usages of a symbol
- `goToDefinition` - Navigate to symbol definition
- `workspaceSymbol` - Search symbols across workspace

### Loop Step 5.5: Run Tests (for implementation beads)

**Skip if** bead has label `type:test` or `no-test:*` (test beads and non-testable beads don't run test validation)

**For implementation beads (label `type:impl`):**

1. **Find associated test file:**
   - Parse bead description for "Test File:" path
   - Or derive from implementation file: `src/foo.ts` → `src/foo.test.ts`

2. **Run tests:**

   ```bash
   npx vitest run <test-file-path> --reporter=verbose
   ```

3. **Validate result:**
   - Exit code 0 → Tests pass → Proceed to close bead
   - Exit code != 0 → Tests fail → DO NOT close bead

4. **If tests fail:**
   - Report which tests failed
   - Keep bead in `in_progress` status
   - Ask user how to proceed (Loop mode)
   - Mark as blocked (Auto Loop mode)

### Test Failure Handling (Loop mode)

```
Use AskUserQuestion with:
- question: "Tests failed for <bead-id>. What would you like to do?"
- header: "Test Fail"
- options:
  - label: "Review and fix"
    description: "Stay on this bead and fix the implementation"
  - label: "Skip (mark blocked)"
    description: "Mark bead as blocked and continue to next"
  - label: "Stop"
    description: "Stop the loop and review manually"
```

### Loop Step 6: Complete the Task

**Only proceed if tests pass (for impl beads) or bead is test/non-testable.**

```bash
bd close <id> --reason "Done: <brief summary>"
```

### Loop Step 7: Update OpenSpec (if applicable)

**IMPORTANT**: If working on an OpenSpec change (label starts with `openspec:`):

1. Find the task number in the bead's **Context Chain** section:
   ```
   **Task**: X.X from tasks.md
   ```

2. If Context Chain is missing or malformed, **skip this step** and log a warning:
   ```
   WARNING: Could not find task number in Context Chain, skipping OpenSpec update
   ```
   The bead can still complete successfully.

3. Edit `openspec/changes/<name>/tasks.md` to mark that task complete:
   - Find the line starting with: `- [ ] X.X` (matching the task number)
   - Change `[ ]` to `[x]` (keep everything else the same)

   Example:
   - Before: `- [ ] 1.1 Create useResponsive hook`
   - After:  `- [x] 1.1 Create useResponsive hook`

4. This applies to ALL beads with `openspec:*` label (including test beads).

### Loop Step 8: Repeat or Pause

**In Loop mode (step):** Use AskUserQuestion to pause for human control.

**In Auto Loop mode:** Continue to next ready bead immediately.

**Before pausing, output execution status:**

```
===============================================================
BEAD COMPLETED: <bead-id>
===============================================================

Progress: N/M beads complete

EXECUTION ORDER (remaining):
  Next → <bead-id>: <title> (P0)
  Then → <bead-id>: <title> (P0)
  Then → <bead-id>: <title> (P1, blocked until P0 done)
===============================================================
```

**Then use AskUserQuestion (Loop mode only):**

```
Use AskUserQuestion with:
- question: "Bead complete. Next: <next-bead-id> (<title>). Continue?"
- header: "Next step"
- options:
  - label: "Continue (Recommended)"
    description: "Proceed to <next-bead-id>: <title>"
  - label: "Stop"
    description: "End the implementation loop here"
  - label: "<other-bead-id>"
    description: "Skip to: <other-bead-title>"
```

Based on the response:

- **Continue**: Proceed to next in execution order
- **Stop**: End the loop and report progress
- **Specific bead ID**: Work on that bead next

### Loop Completion

When no ready tasks remain, say: **"All beads complete"**

---

## Report Results

```
===============================================================
IMPLEMENTATION COMPLETE
===============================================================

Mode: <Swarm|Loop|Auto Loop>
Epic: <epic-title> (if specified)
Label: <label>

Results:
  Test Beads: X completed (test files created)
  Impl Beads: Y completed (tests passing)
  Non-Testable: Z completed
  Blocked: W beads (tests failed or other issues)
  Remaining: R beads

Test Summary:
  Total Test Files: N
  Tests Passing: P
  Tests Failing: F

===============================================================
```

## Progress Visualization

Press `ctrl+t` at any time to see:

- Number of active workers (swarm)
- Tasks in progress
- Completed tasks
- Pending tasks

## Context Recovery

If you lose track:

```bash
bd ready                        # See what's next
bd list --status in_progress    # Find current work
bd show <id>                    # Full task details
```

## Git Policy

**NEVER push to git.** Do not run `git push`, `bd sync`, or any command that pushes to remote. The user will push manually when ready.

## Error Handling

| Scenario               | Action                                                                         |
| ---------------------- | ------------------------------------------------------------------------------ |
| No epics found         | "No epics found. Run `/create-beads` to create beads from an OpenSpec change."        |
| No ready beads         | "No ready beads. All beads may be completed or blocked."                       |
| Vitest not configured  | "WARNING: Test validation will be skipped. Install: `npm install -D vitest`"   |
| Tests fail (impl bead) | Mark bead as blocked, report failed tests, ask user (Loop) or continue (Swarm) |
| Worker fails (swarm)   | Mark bead as blocked, report error, continue with others                       |
| All beads blocked      | Stop, report blocking issues                                                   |
| Max iterations reached | Loop stops automatically; resume with `/implement-beads` to continue                 |

## Stopping

- Say "All beads complete" when done
- Run `/cancel-implementation` to stop early
- In Loop mode: select "Stop" at the pause prompt

## Example Usage

```bash
# Interactive mode (select epic and mode)
/implement-beads

# With specific label (skip epic selection)
/implement-beads --label openspec:add-auth

# With specific mode (skip mode selection)
/implement-beads --mode swarm

# Full specification
/implement-beads --label openspec:add-auth --mode swarm --workers 5 --model opus

# Auto loop mode
/implement-beads --label openspec:refactor --mode auto
```

## When to Use Loop vs Swarm

| Situation                               | Use                |
| --------------------------------------- | ------------------ |
| Tasks depend heavily on each other      | Loop or Auto Loop  |
| Want to review each step                | Loop               |
| Many independent tasks                  | Swarm              |
| Want maximum speed                      | Swarm              |
| Debugging issues                        | Loop               |
| Large refactoring with phases           | Swarm              |
| Want to fix test failures interactively | Loop               |
| Tests are stable and expected to pass   | Swarm or Auto Loop |
