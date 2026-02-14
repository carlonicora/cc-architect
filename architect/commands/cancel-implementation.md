---
description: "Gracefully stop any active implementation (loop, swarm, or agent teams)"
allowed-tools: ["Bash", "Task", "TaskOutput"]
argument-hint: ""
---

# Cancel Command

Gracefully stop any active implementation while preserving progress.

**Works with:** `/implement-beads` (Swarm, Loop, Auto Loop) and `/implement-team-beads` (Agent Teams)

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

Note: Swarm workers and Agent Teams teammates may complete their
current task before stopping.
Check status in a moment with: bd list

For Agent Teams: tell the lead to shut down all teammates and
clean up the team.

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
  /implement-beads                # Swarm, Loop, Auto Loop
  /implement-team-beads           # Agent Teams
```

## Data Preservation

- **Completed tasks**: Remain closed
- **In-progress tasks**: Remain in_progress (swarm workers may complete)
- **Pending tasks**: Remain pending

The beads database preserves all state. You can resume at any time with `/implement-beads` or `/implement-team-beads`.

## Cancellation by Mode

| Aspect | Loop/Auto Loop | Swarm | Agent Teams |
|--------|----------------|-------|-------------|
| Stops | Single foreground agent | Multiple background workers | Lead shuts down teammates, cleans up team |
| Immediate | Yes | Workers may finish current task | Teammates may finish current task |
| State | Deterministic | Check `bd list` after a moment | Check `bd list` after a moment |

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

Note: Swarm workers and Agent Teams teammates may complete their
current task before stopping.

To check status:
  bd list

To resume:
  /implement-beads                # Swarm, Loop, Auto Loop
  /implement-team-beads           # Agent Teams

===============================================================
```
