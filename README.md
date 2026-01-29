# Architect for Claude Code

**Plan → Spec → Beads → Execute.** Persistent task management that survives context compaction and session boundaries.

## The Problem

Claude Code is powerful, but without structure it can:

- Start coding before understanding the full picture
- Lose track of what's done when context gets long
- Say "done" when tests are still failing
- Hallucinate on large features that exceed context

**The Solution:** Break work into self-contained beads with full implementation details.

```bash
/architect:openspec "Add user authentication with JWT"
/architect:review                   # Interactive: validate specs before implementation
/architect:beads                    # Interactive: select from available specs
/architect:implement                # Interactive: select epic and execution mode
```

## Install

### cc-architect Plugin

```bash
# Add to marketplace
/plugin marketplace add carlonicora/cc-architect

# Install the architect plugin
/plugin install architect@cc-architect
```

### Dependencies

| Tool                                               | Install                                                                          | Required For                               |
| -------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------ |
| [OpenSpec](https://github.com/Fission-AI/OpenSpec) | See [OpenSpec installation](https://github.com/Fission-AI/OpenSpec#installation) | `/architect:openspec`, `/architect:review` |
| [Beads](https://github.com/steveyegge/beads)       | `brew tap steveyegge/beads && brew install bd`                                   | `/architect:beads`, `/architect:implement` |

## Commands

| Command                                  | Purpose                  | Interactive                 |
| ---------------------------------------- | ------------------------ | --------------------------- |
| `/architect:openspec [plan\|task]`       | Create OpenSpec proposal | Optional input              |
| `/architect:review [spec-path]`          | Validate and edit specs  | Yes - sprint-meeting style  |
| `/architect:beads [spec-path]`           | Convert spec(s) to beads | Yes - lists available specs |
| `/architect:implement [--label\|--mode]` | Execute beads            | Yes - select epic and mode  |
| `/architect:cancel`                      | Stop active execution    | No                          |

## Workflow

```
PLANNING                              EXECUTION
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│ 1. /architect:openspec <task>   │──▶│ 3. /architect:beads             │
│ 2. /architect:review            │   │ 4. /architect:implement         │
└─────────────────────────────────┘   └─────────────────────────────────┘
```

### Stage 1: Create Proposal

```bash
/architect:openspec "Add billing integration with Stripe"
```

Creates an OpenSpec proposal at `openspec/changes/<id>/` with:

- `proposal.md` - Overview and scope
- `design.md` - Reference implementation code
- `tasks.md` - Task breakdown with exit criteria

### Stage 2: Review Spec

```bash
# Interactive mode (recommended):
/architect:review
# Lists available OpenSpec changes and walks through validation

# Direct mode:
/architect:review openspec/changes/billing/
```

**Sprint-meeting-style validation** that walks through your spec section by section:

1. **Proposal Review** — Why, What Changes, Impact, Success Criteria, Out of Scope
2. **Requirements Review** — For each requirement:
   - **Business Logic**: Is the requirement correct?
   - **Architecture**: Is the technical approach sound?
   - **Development**: Are implementation details correct?
3. **Design Review** — Reference Implementation, Migration Patterns, Risk Analysis
4. **Tasks Review** — Phases, Exit Criteria

**Answer "Yes" to continue, "No" to discuss and edit.** Changes are applied immediately with cascading updates to related files.

**Why review?** Editing specs is cheap; debugging bad beads is expensive. Catch issues before code is written.

### Stage 3: Import to Beads

```bash
# Interactive mode (recommended):
/architect:beads
# Lists available OpenSpec changes and asks which to convert

# Direct mode:
/architect:beads openspec/changes/billing/

# Multiple specs (from auto-decomposed proposal):
/architect:beads openspec/changes/billing-backend/ openspec/changes/billing-frontend/
```

**Auto-decomposition:** Beads >200 lines are automatically split—parent marked `decomposed`, child sub-beads created.

### Stage 4: Execute

```bash
# Interactive mode (recommended):
/architect:implement
# Lists available epics, then asks execution mode: Swarm, Loop, or Auto Loop

# With specific label:
/architect:implement --label openspec:billing

# With specific mode:
/architect:implement --mode swarm --workers 5
/architect:implement --mode loop
/architect:implement --mode auto
```

**Cancel execution:**

```bash
/architect:cancel
```

## Execution Modes

| Mode          | Description                   | Best For                         |
| ------------- | ----------------------------- | -------------------------------- |
| **Swarm**     | Parallel workers (default: 3) | Independent tasks, maximum speed |
| **Loop**      | Sequential with pauses        | Interdependent tasks, debugging  |
| **Auto Loop** | Sequential, no pauses         | Hands-off sequential execution   |

**Choose Loop/Auto Loop when:**

- Tasks depend heavily on each other
- You want to review each step (Loop mode)
- Debugging issues

**Choose Swarm when:**

- Many independent tasks
- Want maximum speed
- Large refactoring with parallel phases

## Progress Visualization

Press `ctrl+t` at any time during execution to see:

- Active workers (swarm)
- Tasks in progress
- Completed tasks
- Pending tasks

## What are Beads?

Atomic, self-contained task units in a local graph database. Unlike session-scoped plans, beads **persist** across sessions.

| Component                | Purpose                             |
| ------------------------ | ----------------------------------- |
| Requirements             | Full requirements copied verbatim   |
| Reference Implementation | 20-400+ lines based on complexity   |
| Migration Pattern        | Exact before/after for file edits   |
| Exit Criteria            | Specific verification commands      |
| Dependencies             | `depends_on`/`blocks` relationships |

**Key insight:** When context compacts, `bd ready` always shows what's next. Each bead has everything needed—no reading other files.

### Essential `bd` Commands

| Command                               | Action                      |
| ------------------------------------- | --------------------------- |
| `bd ready`                            | List tasks with no blockers |
| `bd show <id>`                        | Show full task details      |
| `bd update <id> --status in_progress` | Start working on task       |
| `bd close <id> --reason "Done"`       | Complete task               |
| `bd list --status in_progress`        | Find current work           |

## The Self-Contained Bead Rule

Each bead must be implementable with ONLY its description.

| Bad             | Good                               |
| --------------- | ---------------------------------- |
| "See design.md" | FULL code (50-200+ lines) in bead  |
| "Run tests"     | `npm test -- stripe-price`         |
| "Update entity" | File + line numbers + before/after |

## Best Practices

1. **Review specs before beads** — Use `/architect:review` to validate business logic, architecture, and implementation details
2. **Exit criteria are non-negotiable** — Not "tests pass" but exact commands: `npm test -- auth`
3. **Use Loop mode for complex tasks** — Prevents quality degradation
4. **Use Swarm for independent tasks** — Maximizes parallelism
5. **Review beads** — `bd list -l "openspec:<name>"` then `bd show <id>` to verify code snippets

## Stealth Mode

For brownfield development, keep tracking files out of git:

```bash
bd init --stealth    # Adds .beads/ to .gitignore
```

## Context Recovery

If you lose track during execution:

```bash
bd ready                        # See what's next
bd list --status in_progress    # Find current work
bd show <id>                    # Full task details
```

## Resources

- [Beads GitHub](https://github.com/steveyegge/beads)
- [OpenSpec GitHub](https://github.com/Fission-AI/OpenSpec)
- [Installing bd](https://github.com/steveyegge/beads/blob/main/docs/INSTALLING.md)

## License

MIT
