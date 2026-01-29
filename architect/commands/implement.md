---
description: "Execute beads using loop (sequential) or swarm (parallel) mode"
argument-hint: "[--label <label>] [--epic <id>] [--mode <swarm|loop|auto>]"
allowed-tools: ["Task", "TaskOutput", "Read", "TodoWrite", "Bash", "Edit", "AskUserQuestion"]
---

# Implement Command

Execute beads using your choice of execution strategy.

**IMPORTANT**: This command combines loop (sequential) and swarm (parallel) execution modes with interactive selection.

## Execution Modes

| Mode | Description | Best For |
|------|-------------|----------|
| **Swarm** | Parallel workers | Independent beads, maximum speed |
| **Loop** | Sequential with step-by-step confirmation | Interdependent tasks, debugging |
| **Auto Loop** | Sequential without confirmation | Hands-off sequential execution |

## Arguments

- `--label <label>`: Filter beads by label (e.g., `--label openspec:my-change`) - skips epic selection
- `--epic <id>`: Execute beads for a specific epic - skips epic selection
- `--mode <swarm|loop|auto>`: Skip mode selection prompt
- `--workers N`: Maximum concurrent workers for swarm mode (default: 3)
- `--model MODEL`: Worker model for swarm - `haiku`, `sonnet`, or `opus` (default: opus)

## Instructions

### Step 1: Validate Environment

```bash
# Check bd installed
bd version

# Check initialized
bd ready &>/dev/null && echo "OK" || echo "Run: bd init --stealth"
```

### Step 2: Parse Arguments

Parse `$ARGUMENTS` for flags:
- `--label <value>` → skip epic selection, use this label
- `--epic <id>` → skip epic selection, use this epic
- `--mode <swarm|loop|auto>` → skip mode selection
- `--workers <N>` (default: 3)
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
- No epics found → "No epics found. Run `/beads` to create beads from an OpenSpec change."
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

       ## Task Description
       <full bead description from bd show>

       ## Instructions
       1. Read the files mentioned
       2. Make the required changes (use LSP tools for TypeScript: documentSymbol, findReferences, goToDefinition)
       3. Run verification commands in exit criteria
       4. If this bead has an `openspec:` label: Edit `openspec/changes/<name>/tasks.md` to mark the corresponding task as `[x]`
       5. Report success or failure

       When complete, output: "BEAD COMPLETE: <id>"
       If blocked, output: "BEAD BLOCKED: <id> - <reason>"
   ```

3. Track worker assignment: `worker_id → bead_id`

### Swarm Step 4: Monitor Completion

Wait for workers to complete using TaskOutput:

1. Poll each running worker with `TaskOutput(task_id, block=false)`
2. When worker completes:
   - Parse output for "BEAD COMPLETE" or "BEAD BLOCKED"
   - If complete: `bd close <id> --reason "Done"`
   - If blocked: Report issue, mark as blocked
3. Update dependency graph (remove completed from blockedBy lists)
4. Identify newly unblocked beads
5. Spawn replacement workers for new ready beads

### Swarm Step 5: Continue Until Done

Repeat until:
- All beads are completed, OR
- No more work can be done (all remaining are blocked)

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

### Loop Step 6: Complete the Task

```bash
bd close <id> --reason "Done: <brief summary>"
```

### Loop Step 7: Update OpenSpec (if applicable)

**IMPORTANT**: If working on an OpenSpec change (label starts with `openspec:`):

Edit `openspec/changes/<name>/tasks.md` to mark that task complete:

```markdown
# Before
- [ ] Task description here

# After
- [x] Task description here
```

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
  Completed: X beads
  Blocked: Y beads (if any)
  Remaining: Z beads

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

| Scenario | Action |
|----------|--------|
| No epics found | "No epics found. Run `/beads` to create beads from an OpenSpec change." |
| No ready beads | "No ready beads. All beads may be completed or blocked." |
| Worker fails (swarm) | Mark bead as blocked, report error, continue with others |
| All beads blocked | Stop, report blocking issues |
| Max iterations reached | Loop stops automatically; resume with `/implement` to continue |

## Stopping

- Say "All beads complete" when done
- Run `/cancel` to stop early
- In Loop mode: select "Stop" at the pause prompt

## Example Usage

```bash
# Interactive mode (select epic and mode)
/implement

# With specific label (skip epic selection)
/implement --label openspec:add-auth

# With specific mode (skip mode selection)
/implement --mode swarm

# Full specification
/implement --label openspec:add-auth --mode swarm --workers 5 --model opus

# Auto loop mode
/implement --label openspec:refactor --mode auto
```

## When to Use Loop vs Swarm

| Situation | Use |
|-----------|-----|
| Tasks depend heavily on each other | Loop or Auto Loop |
| Want to review each step | Loop |
| Many independent tasks | Swarm |
| Want maximum speed | Swarm |
| Debugging issues | Loop |
| Large refactoring with phases | Swarm |
