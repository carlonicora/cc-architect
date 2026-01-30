---
allowed-tools: Task, TaskOutput, Bash, Read, Glob, AskUserQuestion
argument-hint: "[spec-path]"
description: Convert OpenSpec specs to self-contained Beads (project)
---

# Beads Creator

Convert OpenSpec specification(s) into self-contained Beads issues.

**IMPORTANT**: Each bead must be SELF-CONTAINED - implementable with ONLY its description. Never reference external specs without copying the content.

## Why Beads?

Beads provide structured, trackable work items that integrate with the OpenSpec workflow.

Key advantages:
- **Self-Contained** - Each bead includes all context needed for implementation
- **Traceable** - Dual back-references link beads to specs and plans
- **Parallelizable** - Independent beads can be worked on simultaneously

## What Good Beads Include

1. **Context Chain**
   - Spec reference path
   - Plan reference path
   - Task reference from tasks.md

2. **Requirements**
   - Full text copied from spec (never "see spec")
   - Code examples from design.md
   - Exact file paths

3. **Exit Criteria**
   - Specific test commands
   - Build verification steps
   - Acceptance criteria checkboxes

4. **TDD Structure** (for testable tasks)
   - Test beads (type:test) - create test files first
   - Impl beads (type:impl) - depend on test beads
   - Non-testable beads (no-test:*) - auto-detected config/type changes

## Arguments

- **No argument**: Interactive mode - lists available OpenSpec changes and asks which to convert
- **Spec path**: Direct mode - uses the provided path(s)
- **Multiple paths**: `openspec/changes/<name-1>/ openspec/changes/<name-2>/`

## Instructions

### Step 1: Validate Environment

```bash
# Check bd installed
bd version

# Check initialized (use --stealth for brownfield projects)
bd ready &>/dev/null && echo "OK" || echo "Run: bd init --stealth"
```

**Note:** Use `bd init --stealth` for brownfield development. This keeps `.beads/` out of git so you don't pollute existing repos with tracking files.

### Step 2: Determine Mode

**If $ARGUMENTS contains a path** (e.g., `openspec/changes/...`):
- Use direct mode - skip to Step 3

**If $ARGUMENTS is empty**:
- Use interactive mode - continue below

### Step 2a: List Available OpenSpec Changes (Interactive Mode)

```bash
# List all directories in openspec/changes/ excluding archive/
ls -d openspec/changes/*/ 2>/dev/null | grep -v '/archive/' | while read dir; do
  change_name=$(basename "$dir")
  # Get brief description from proposal.md (first line after "# Change:")
  desc=$(grep -A1 "^# Change:" "$dir/proposal.md" 2>/dev/null | tail -1 | head -c 60)
  echo "$change_name: $desc"
done
```

**Present Available Changes:**

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Select an OpenSpec change to convert to beads:"
- header: "OpenSpec"
- options:
  - label: "<change-name-1>"
    description: "<brief overview from proposal.md>"
  - label: "<change-name-2>"
    description: "<brief overview from proposal.md>"
  ... (one option per available change, max 4)
```

**Handle Selection:**
- If user selects a change name → set `SPEC_PATH=openspec/changes/<selected>/`
- If user selects "Other" → use their custom input as path

**Edge Cases:**
- No changes found → "No OpenSpec changes found. Run `/openspec` first."
- All in archive → "All changes archived. Run `/openspec` to create new."

### Step 3: Parse and Validate Specs

```bash
# Split arguments into array of spec paths
SPEC_PATHS=($ARGUMENTS)

# Validate each spec
for spec_path in "${SPEC_PATHS[@]}"; do
  if [ -f "$spec_path/proposal.md" ]; then
    echo "✓ $spec_path validated"
  else
    echo "ERROR: $spec_path is not a valid OpenSpec directory (missing proposal.md)"
    exit 1
  fi
done

echo "Total specs: ${#SPEC_PATHS[@]}"
```

### Step 4: Read Spec Files

**For single spec:**
```bash
cat $ARGUMENTS/proposal.md
cat $ARGUMENTS/tasks.md
cat $ARGUMENTS/design.md 2>/dev/null || true
find $ARGUMENTS/specs -name "*.md" -exec cat {} \; 2>/dev/null || true
```

**For multiple specs (decomposed):**
```bash
for spec_path in "${SPEC_PATHS[@]}"; do
  echo "=== Spec: $spec_path ==="
  cat $spec_path/proposal.md
  cat $spec_path/tasks.md
  cat $spec_path/design.md 2>/dev/null || true
  find $spec_path/specs -name "*.md" -exec cat {} \; 2>/dev/null || true
done
```

### Step 5: Launch Agent

Launch `beads-default`:

```
Convert OpenSpec(s) to self-contained beads.

Spec Count: <1 or N>
Spec Paths: <path(s)>
Change Names: <name(s)>

## Spec Content

<content of all spec files>

## Key Principle

Each bead must be SELF-CONTAINED:
- Copy requirements verbatim (never "see spec")
- Include code examples
- Include exact file paths
- Include acceptance criteria

For single spec: Create epic first, then child tasks with --parent.
For multiple specs: Create one epic per spec, respect depends_on between specs.
```

Use `subagent_type: "beads-default"` and `run_in_background: true`.

### Step 6: Report Result

**For single spec:**
```
===============================================================
BEADS CREATED
===============================================================

Epic: <id>
Tasks: <count> (test: N, impl: M, non-testable: P)
Ready: <count>

TDD Structure:
  Test Beads: N (create first, must compile)
  Impl Beads: M (depend on test beads, must pass tests)
  Non-Testable: P (no test requirement)

Complexity Assessment: <trivial|small|medium|large|huge>
Decomposition Strategy: <single-bead|multi-bead|hierarchical>

## Quality Report
| Metric | Value | Status |
|--------|-------|--------|
| Total beads | N | OK |
| Test beads | N | OK |
| Impl beads | N | OK |
| Test coverage | 100% | OK |
| Avg lines | N | OK |
| Independence | N% | OK |
| Max chain | N | OK |

Recommendation: PROCEED

Next Steps:
1. Review: bd list -l "openspec:<name>"
2. Execute: /implement
   - Test beads run first (create test files)
   - Impl beads run after (implementation + test execution)

===============================================================
```

**For multiple specs (decomposed):**
```
===============================================================
BEADS CREATED (FROM DECOMPOSED PROPOSALS)
===============================================================

Specs Processed: N

| Spec | Epic ID | Test Beads | Impl Beads | Ready |
|------|---------|------------|------------|-------|
| add-auth-backend | epic-001 | 3 | 2 | 3 |
| add-auth-frontend | epic-002 | 2 | 2 | 0 |
| add-auth-integration | epic-003 | 1 | 2 | 0 |

Cross-Spec Dependencies:
  epic-001 (add-auth-backend) → blocks → epic-002 (add-auth-frontend)
  epic-002 (add-auth-frontend) → blocks → epic-003 (add-auth-integration)

TDD Structure:
  Total Test Beads: 6 (create first, must compile)
  Total Impl Beads: 6 (depend on test beads, must pass tests)
  Total Beads: 12
  Ready Now: 3 (test beads from add-auth-backend)

## Quality Report
| Metric | Value | Status |
|--------|-------|--------|
| Total beads | 12 | OK |
| Test beads | 6 | OK |
| Impl beads | 6 | OK |
| Test coverage | 100% | OK |
| Avg lines | 85 | OK |
| Independence | 65% | OK |
| Max chain | 3 | OK |

Recommendation: PROCEED

Next Steps:
1. Review: bd list
2. Execute: /implement
   - Test beads run first (create test files)
   - Impl beads run after (implementation + test execution)

===============================================================
```

## Self-Contained Bead Rule

Each bead must be implementable with ONLY its description. Include **dual back-references** for context recovery.

**BAD:**
```
bd create "Update auth" -t task
```

**GOOD - Test Bead (TDD):**
```
bd create "Test: Add JWT validation" -t task -p 2 \
  --parent <epic-id> \
  -l "openspec:add-auth" \
  -l "type:test" \
  -d "## Test Bead

**Validation**: Complete when test file exists and compiles.

## Test File to Create
**Path**: src/auth.test.ts

## Full Test Code
\`\`\`typescript
import { describe, it, expect } from 'vitest'
// FULL test code here - copy-paste ready
\`\`\`

## Exit Criteria
- [ ] Test file created
- [ ] \`npx vitest typecheck\` passes"
```

**GOOD - Impl Bead (depends on test):**
```
bd create "Add JWT validation" -t task -p 2 \
  --parent <epic-id> \
  -l "openspec:add-auth" \
  -l "type:impl" \
  -d "## Context Chain

**Spec Reference**: openspec/changes/add-auth/specs/auth/spec.md
**Test Bead ID**: <test-bead-id>
**Test File**: src/auth.test.ts

## Requirements
<copied from spec - FULL text>

## Reference Implementation
<FULL code from design.md>

## Exit Criteria
- [ ] \`npx vitest run src/auth.test.ts\` passes
- [ ] TypeScript compiles

## Files
- src/auth.ts"
```

**Litmus test:** Could someone implement this with ONLY the bead description? If not, add more context.

## Auto-Decomposition

The agent **automatically** decomposes large beads - no user prompt needed.

| Trigger | Threshold | Action |
|---------|-----------|--------|
| Bead size | >200 lines | Auto-create sub-beads |
| Complexity | Multiple responsibilities | Auto-split by concern |
| Independence | <60% score | Auto-restructure dependencies |

### Auto-Decomposition Process

When beads exceed thresholds, the agent automatically:

1. **Identifies large beads** from quality metrics
2. **Creates child sub-beads** with `--parent`:
   ```bash
   bd update <parent-id> --status decomposed
   bd create "Sub-task 1" -t task --parent <parent-id> -l "openspec:<name>" -d "<self-contained>"
   bd create "Sub-task 2" -t task --parent <parent-id> -l "openspec:<name>" -d "<self-contained>"
   ```
3. **Each sub-bead is self-contained** - copies context from parent
4. **Reports decomposition** in output

### Decomposition Output

```
===============================================================
BEADS CREATED (AUTO-DECOMPOSED)
===============================================================

Original Beads: 5
Decomposed: 2 (too large)
New Sub-Beads: 6

Decomposition Details:
  • bead-003 "Implement auth handler" → 3 sub-beads
    - bead-003a "Add JWT validation middleware"
    - bead-003b "Implement token refresh logic"
    - bead-003c "Add session management"
  • bead-005 "Add API endpoints" → 3 sub-beads
    - bead-005a "POST /auth/login endpoint"
    - bead-005b "POST /auth/logout endpoint"
    - bead-005c "GET /auth/me endpoint"

Total Beads Now: 9
Ready: 4

Next:
- Execute: /implement
===============================================================
```

## Workflow Diagram

Works with plans from **any planner**:

```
/plan-creator <task>           ┐
/bug-plan-creator <error>      ├─▶ .claude/plans/*-plan.md
/code-quality-plan-creator     ┘
    │
    ▼
/openspec <plan>
    │
    ├─── [single proposal] ─────────▶ openspec/changes/<id>/
    │                                       │
    └─── [decomposed: multiple] ────▶ openspec/changes/<id-1>/
                                      openspec/changes/<id-2>/
                                      openspec/changes/<id-3>/
    │                                       │
    ▼                                       ▼
/beads (interactive)            /beads <spec-1> <spec-2> <spec-3>
    │                                       │
    └──▶ /implement                         └──▶ /implement
```

## Step Mode (Default)

Step mode pauses after creating beads for human review.

**After creating beads, you MUST use AskUserQuestion to pause.**

```
Use AskUserQuestion with:
- question: "Beads created. What would you like to do?"
- header: "Next step"
- options:
  - label: "Implement (Recommended)"
    description: "Proceed to /implement to execute beads"
  - label: "Stop"
    description: "End here for manual review"
```

Use `--auto` flag to skip pauses and proceed directly to /implement.

## Git Policy

**NEVER push to git.** Do not run `git push`, `bd sync`, or any command that pushes to remote. The user will push manually when ready.

## Error Handling

| Scenario | Action |
|----------|--------|
| bd not installed | "Install bd: https://github.com/steveyegge/beads" |
| bd not initialized | "Run: bd init" |
| Path not found | Report error |

## Example Usage

```bash
# Interactive mode (lists available changes)
/beads

# Direct mode with single spec
/beads openspec/changes/add-auth/

# Multiple decomposed specs
/beads openspec/changes/add-auth-backend/ openspec/changes/add-auth-frontend/

# Auto mode (skip review pause)
/beads openspec/changes/add-auth/ --auto
```
