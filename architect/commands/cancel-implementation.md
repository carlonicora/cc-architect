---
description: "Gracefully stop any active implementation (loop or swarm)"
allowed-tools: ["Bash", "Task", "TaskOutput"]
argument-hint: ""
---

# Cancel Command

Gracefully stop any active implementation while preserving progress.

**Works with:** `/implement-beads` command (all modes: Swarm, Loop, Auto Loop)

## Instructions

### Step 1: Check Beads Status

```bash
bd list --status in_progress
```

This shows beads currently being worked on.

### Step 2: Report Status

Report current state:

```
===============================================================
IMPLEMENTATION CANCELLED
===============================================================

In-Progress Beads: N
  - <id>: <title>
  ...

Completed Beads: M
Pending Beads: P

Progress is preserved. Completed beads remain closed.
In-progress beads remain in_progress.

Note: Swarm workers may complete their current task before stopping.
Check status in a moment with: bd list

===============================================================
```

### Step 3: Recovery Instructions

Provide recovery commands:

```
To check current status:
  bd list                         # See all beads
  bd ready                        # See what's ready next
  bd list --status in_progress    # Find current work

To resume:
  /implement-beads
```

## Data Preservation

- **Completed tasks**: Remain closed
- **In-progress tasks**: Remain in_progress (swarm workers may complete)
- **Pending tasks**: Remain pending

The beads database preserves all state. You can resume at any time with `/implement-beads`.

## Swarm vs Loop Cancellation

| Aspect | Loop/Auto Loop | Swarm |
|--------|----------------|-------|
| Stops | Single foreground agent | Multiple background workers |
| Immediate | Yes | Workers may finish current task |
| State | Deterministic | Check `bd list` after a moment |

## Example Output

```
===============================================================
IMPLEMENTATION CANCELLED
===============================================================

In-Progress Beads: 2
  - task-004: Add login endpoint
  - task-005: Add logout endpoint

Completed Beads: 3
Pending Beads: 4

Progress is preserved. Completed beads remain closed.
In-progress beads remain in_progress.

Note: Swarm workers may complete their current task before stopping.

To check status:
  bd list

To resume:
  /implement-beads

===============================================================
```
