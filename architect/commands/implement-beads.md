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

**CRITICAL: DO NOT EXIT after spawning workers. You MUST stay in the monitoring loop and call TaskOutput for EACH active worker. The loop only ends when ALL beads are complete or blocked with zero active workers.**

**Initialize tracking before entering the loop:**

- `active_workers`: map of `task_id → bead_id` for all spawned workers
- `completed_count`: 0
- `blocked_count`: 0
- `total_beads`: count of all beads for this label

---

### ⛔ STOP - MANDATORY BLOCKING WAIT ⛔

**You have spawned background workers. You MUST NOT continue until they complete.**

**EXECUTE NOW - Call TaskOutput for EACH worker:**

Take your first `task_id` from `active_workers` and execute this tool call:

```
TaskOutput(task_id="<first_task_id>", block=true, timeout=180000)
```

Wait for the result. Then call TaskOutput for the next worker. Repeat for ALL workers.

**Example:** If you spawned 3 workers with task_ids `abc123`, `def456`, `ghi789`:

1. Call `TaskOutput(task_id="abc123", block=true, timeout=180000)` - WAIT for result
2. Call `TaskOutput(task_id="def456", block=true, timeout=180000)` - WAIT for result
3. Call `TaskOutput(task_id="ghi789", block=true, timeout=180000)` - WAIT for result

**You MUST make these TaskOutput tool calls NOW. Do not just print "Waiting..." and exit.**

---

### After Each Worker Completes

When TaskOutput returns, parse the output:

- If output contains `"BEAD COMPLETE: <id>"`:
  - Run: `bd close <id> --reason "Done"`
  - Worker succeeded

- If output contains `"BEAD BLOCKED: <id>"`:
  - Run: `bd update <id> --status blocked`
  - Worker failed

Remove the completed worker from your tracking.

---

### After ALL Workers Complete

1. **Check for newly ready beads:**
   ```bash
   bd ready -l "<label>" --json
   ```

2. **If there are newly ready beads:** Spawn new workers (up to `--workers` limit) and go back to "STOP - MANDATORY BLOCKING WAIT" above.

3. **If no more ready beads:** Report final results and exit.

---

### Exit Conditions (ONLY exit when ALL of these are true)

- [ ] ALL TaskOutput calls have been made and returned
- [ ] `active_workers` is empty
- [ ] No beads with `pending` status have satisfied dependencies
- [ ] All beads are either `completed` or `blocked`

**If you have active workers and haven't called TaskOutput for each one, GO BACK and call TaskOutput NOW.**

---

## MODE: LOOP / AUTO LOOP

Execute beads iteratively until all ready tasks are complete.

---

### ⛔ CRITICAL: NEVER IMPLEMENT DIRECTLY ⛔

**You MUST delegate ALL bead work to the `architect:bead-worker` agent.**

**PROHIBITED ACTIONS in Loop mode:**
- ❌ Reading implementation files directly (let worker do it)
- ❌ Using Edit tool on source code (let worker do it)
- ❌ Running tests directly (let worker do it)
- ❌ Modifying OpenSpec tasks.md (let worker do it)

**REQUIRED ACTIONS in Loop mode:**
- ✅ Use `bd` commands to find/show/update beads
- ✅ Spawn `architect:bead-worker` via Task tool
- ✅ Wait for worker JSON result
- ✅ Close bead based on result

**WHY:** Direct implementation bloats your context window. After a few beads, you become useless. Workers keep their context isolated.

---

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

### Loop Step 5: Delegate to Worker

---

### ⛔ STOP - MANDATORY DELEGATION ⛔

**DO NOT implement this bead yourself. You MUST delegate to a worker agent.**

**If you are tempted to:**
- Read source files → STOP. Spawn the worker instead.
- Edit code directly → STOP. Spawn the worker instead.
- Run tests yourself → STOP. Spawn the worker instead.

**EXECUTE NOW:**

---

1. **Extract bead metadata** from `bd show` output:
   - Bead type: Check for `type:test`, `type:impl`, or `no-test:*` labels
   - OpenSpec label: Look for `openspec:<change-name>` label
   - OpenSpec task: Parse Context Chain section for `**Task**: X.X from tasks.md`

2. **Spawn the worker NOW:**

```
Use Task tool with:
- subagent_type: "architect:bead-worker"
- model: <selected model or opus>
- run_in_background: false  # Wait for result (blocking)
- prompt: |
    Execute this bead task:

    Bead ID: <id>
    Title: <title>
    Bead Type: <test | impl | non-testable>
    OpenSpec Label: <openspec:change-name or "none">
    OpenSpec Task: <task number from Context Chain, e.g., "1.1", or "unknown" if not found>

    ## Task Description
    <full bead description from bd show>

    Execute the bead and return JSON result.
```

3. **Wait for the Task tool to return the worker's JSON result.** Do NOT proceed until you have the result.

### Loop Step 6: Handle Worker Result

Parse the JSON result from worker:

**If status == "COMPLETE":**

1. Close the bead:
   ```bash
   bd close <id> --reason "<summary from worker>"
   ```

2. Output summary (keeps main context minimal):
   ```
   ✓ BEAD COMPLETE: <id> - <summary>
     Files: <files_modified>
     Tests: <test_result>
   ```

**If status == "BLOCKED":**

1. Update bead status:
   ```bash
   bd update <id> --status blocked
   ```

2. Output error:
   ```
   ✗ BEAD BLOCKED: <id> - <error>
   ```

3. **In Loop mode:** Ask user how to proceed (see below)
4. **In Auto Loop mode:** Continue to next bead

### Blocked Bead Handling (Loop mode only)

```
Use AskUserQuestion with:
- question: "Bead blocked: <bead-id>. <error summary>. What would you like to do?"
- header: "Blocked"
- options:
  - label: "Skip and continue"
    description: "Leave bead blocked, proceed to next ready bead"
  - label: "Retry"
    description: "Run the worker again on this bead"
  - label: "Stop"
    description: "Stop the loop and review manually"
```

Based on response:
- **Skip and continue**: Proceed to next ready bead
- **Retry**: Go back to Step 5 for the same bead
- **Stop**: End the loop and report progress

### Loop Step 7: Repeat or Pause

**In Loop mode (step):** Use AskUserQuestion to pause for human control.

**In Auto Loop mode:** Continue to next ready bead immediately.

**Before pausing, show progress (summary only - details stay in worker context):**

```
===============================================================
Progress: N/M beads complete
Blocked: B beads

EXECUTION ORDER (remaining):
  Next → <bead-id>: <title>
  Then → <bead-id>: <title>
===============================================================
```

**Then use AskUserQuestion (Loop mode only):**

```
Use AskUserQuestion with:
- question: "Next: <next-bead-id> (<title>). Continue?"
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

- **Continue**: Proceed to next in execution order (go to Step 1)
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
