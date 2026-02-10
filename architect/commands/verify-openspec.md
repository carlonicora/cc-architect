---
allowed-tools: Task, TaskOutput, Bash, Read, Glob, Grep, AskUserQuestion
argument-hint: "[openspec-change-path]"
description: Pre-implementation verification of OpenSpec against architecture docs
---

# Verify OpenSpec Command

Pre-implementation verification that validates OpenSpec code (in design.md and tests.md) against architecture patterns BEFORE any implementation begins. Uses **4 specialized agents running in parallel** for comprehensive analysis.

**IMPORTANT**: Run this command AFTER `/architect:create-openspec` or `/architect:review-openspec` to verify code quality BEFORE `/architect:implement-beads`.

## Why Verify OpenSpec?

The OpenSpec `design.md` contains Reference Implementation code (50-200+ lines) and `tests.md` contains full test code. If this code violates architecture patterns, it's better to catch it BEFORE implementation than after.

Verification checks:
- **TypeScript Quality**: Type safety, naming, patterns in code blocks
- **Architecture Compliance**: Patterns from docs/architecture/
- **Security**: OWASP vulnerabilities in proposed code
- **Performance**: N+1 queries, memory leaks in proposed code

## Arguments

- **No argument**: Interactive mode - lists available OpenSpec changes
- **Spec path**: Direct mode - uses the provided path (e.g., `openspec/changes/add-auth/`)

## Instructions

### Step 1: Validate Environment

```bash
# Check openspec is installed
openspec version 2>&1 | head -1

# Check openspec is initialized
ls openspec/config.yaml 2>/dev/null && echo "OpenSpec initialized" || echo "Not initialized"

# CRITICAL: Check architecture docs exist
ls docs/architecture/INDEX.md 2>/dev/null && echo "Architecture docs found" || echo "ERROR: Missing docs/architecture/INDEX.md"
```

**Error Handling:**
- If OpenSpec not installed: "Install OpenSpec: https://github.com/Fission-AI/OpenSpec"
- If OpenSpec not initialized: "Run: openspec init"
- If `docs/architecture/INDEX.md` missing: **FAIL** with error:
  ```
  Architecture documentation required for verification.

  Create docs/architecture/INDEX.md with your project's architecture patterns.
  This file should contain:
  - Quick reference table mapping task types to architecture docs
  - Links to detailed architecture documentation
  - Core principles and anti-patterns

  Without architecture docs, the verify-openspec command cannot check compliance.
  ```

### Step 2: Determine Mode

**If $ARGUMENTS contains a path** (e.g., `openspec/changes/...`):
- Use direct mode - skip to Step 3

**If $ARGUMENTS is empty**:
- Use interactive mode - continue below

### Step 2a: List Available OpenSpec Changes (Interactive Mode)

```bash
# List changes that have design.md files
ls -d openspec/changes/*/ 2>/dev/null | while read dir; do
  change_name=$(basename "$dir")
  # Check if design.md and tests.md exist
  has_design=$(test -f "$dir/design.md" && echo "yes" || echo "no")
  has_tests=$(test -f "$dir/tests.md" && echo "yes" || echo "no")
  echo "$change_name: design.md=$has_design, tests.md=$has_tests"
done
```

**Present Available Changes:**

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Select an OpenSpec change to verify:"
- header: "Verify"
- options:
  - label: "<change-name-1>"
    description: "design.md=yes, tests.md=yes"
  - label: "<change-name-2>"
    description: "design.md=yes, tests.md=no"
  ... (one option per available change, max 4)
```

**Handle Selection:**
- If user selects a change name → set `SPEC_PATH=openspec/changes/<selected>/`
- If user selects "Other" → use their custom input as path

**Edge Cases:**
- No changes found → "No OpenSpec changes found. Run `/create-openspec` first."
- No design.md/tests.md → "Selected change has no code to verify."

### Step 3: Read OpenSpec Files and Extract Code Blocks

Read the OpenSpec files:

```bash
# Read design.md
cat $SPEC_PATH/design.md

# Read tests.md
cat $SPEC_PATH/tests.md

# Read proposal.md for context (affected files list)
cat $SPEC_PATH/proposal.md
```

**Extract code blocks:**

Parse design.md and tests.md for fenced code blocks:

```markdown
```typescript
// Code block content here
```
```

For each code block, extract:
- **Language tag** (typescript, javascript, etc.)
- **Block number** (1-indexed, per file)
- **Content** (the code inside)
- **Preceding context** (any comment or section header before the block that hints at filename)

**Format extracted blocks as:**

```
## Code Blocks from design.md

### Block 1 (typescript)
Section: Reference Implementation
Hint: src/auth/service.ts
```typescript
<code content>
```

### Block 2 (typescript)
Section: Migration Pattern - Before
```typescript
<code content>
```

## Code Blocks from tests.md

### Block 1 (typescript)
Hint: src/auth/service.test.ts
```typescript
<code content>
```
```

**Also store:**
- `CHANGE_NAME=$(basename $SPEC_PATH)`
- Parse `proposal.md` for affected files (Impact section)

### Step 4: Read Architecture Docs

```bash
# Read architecture index
cat docs/architecture/INDEX.md

# Read relevant architecture docs based on file types
cat docs/architecture/00-core-principles.md 2>/dev/null || true
cat docs/architecture/backend/*.md 2>/dev/null || true
cat docs/architecture/frontend/*.md 2>/dev/null || true
cat docs/architecture/anti-patterns.md 2>/dev/null || true
```

Store the full content for agent prompts.

### Step 5: Launch 4 Verification Agents in Parallel

**CRITICAL**: Launch ALL agents simultaneously using Task tool with `run_in_background: true`.

Each agent receives:
- Change ID and spec path
- Extracted code blocks from design.md
- Extracted code blocks from tests.md
- Full architecture docs content
- Affected files list from proposal.md

**Agent Launch Template:**

For each of the 4 agents, use:
```
Use Task tool with:
- subagent_type: "architect:verify-openspec-<agent-name>"
- model: opus
- run_in_background: true
- prompt: |
    Verify OpenSpec code for architecture compliance.

    Change ID: <CHANGE_NAME>
    Spec Path: <SPEC_PATH>

    ## Code Blocks from design.md

    <formatted code blocks with language, block number, hints>

    ## Code Blocks from tests.md

    <formatted code blocks with language, block number, hints>

    ## Architecture Docs

    ### INDEX.md
    <full content>

    ### 00-core-principles.md
    <content if exists>

    ### backend/*.md
    <content if exists>

    ### frontend/*.md
    <content if exists>

    ### anti-patterns.md
    <content if exists>

    ## Affected Files (from proposal.md)
    <list from proposal.md Impact section>

    Analyze the code blocks and return findings as JSON.
```

**Launch order** (all in parallel in a single message):
1. `verify-openspec-typescript` - TypeScript quality
2. `verify-openspec-architecture` - Architecture compliance
3. `verify-openspec-security` - Security vulnerabilities
4. `verify-openspec-performance` - Performance issues

Track agent IDs for later collection.

**CRITICAL: DO NOT EXIT after launching agents. You MUST proceed to Step 6 and wait for ALL agents to complete before continuing.**

### Step 6: Wait for All Agents

**CRITICAL: You MUST wait for ALL agents to complete. DO NOT exit or end the conversation after spawning background tasks.**

Use TaskOutput to wait for each agent:

```
For each agent_id:
  Use TaskOutput with:
  - task_id: <agent_id>
  - block: true
  - timeout: 300000  # 5 minutes per agent

  Parse JSON output from agent
  Add findings to consolidated list
```

**Handle agent failures:**
- If agent times out → report partial results, note timeout
- If agent fails → report error, continue with others
- If all agents fail → report environment issue, suggest fixes

**DO NOT proceed to Step 7 until ALL agents have returned results. You MUST call TaskOutput for EACH agent and wait for completion.**

### Step 7: Consolidate and Sort Findings

After all agents complete:

1. **Merge findings** from all 4 agents into single list
2. **Deduplicate** findings with same code_block/line/issue
3. **Sort by severity**: P1 (Critical) → P2 (Important) → P3 (Nice-to-Have)
4. **Group by category** for presentation

**Calculate summary:**
```
total_findings = sum of all agent findings
p1_count = count where severity == "P1"
p2_count = count where severity == "P2"
p3_count = count where severity == "P3"
```

### Step 8: Present Results Summary

```
===============================================================
OPENSPEC VERIFICATION RESULTS: <CHANGE_NAME>
===============================================================

Agents Completed: 4/4
Total Findings: <N> (P1: <X>, P2: <Y>, P3: <Z>)

By Agent:
  verify-openspec-typescript:    <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
  verify-openspec-architecture:  <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
  verify-openspec-security:      <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
  verify-openspec-performance:   <N> findings (P1: <X>, P2: <Y>, P3: <Z>)

===============================================================
```

**If no findings**: Skip to Step 10 (Final Report) with PASSED status.

**If findings exist**: Proceed IMMEDIATELY to Step 9.

### Step 9: Interactive Fix Loop (ONE BY ONE)

**CRITICAL REQUIREMENTS:**

1. **ONE BY ONE**: Present each finding individually. After presenting ONE finding, use AskUserQuestion to get the user's decision. Only after the user responds, move to the next finding. NEVER batch findings or show multiple at once.

2. **NO DIRECT EDITS**: You do NOT have Edit tool access. You MUST NOT use Bash (sed, awk, echo, cat, tee, or any command) to modify files. ALL fixes MUST be delegated to the `architect:fix-finding` agent.

3. **WAIT FOR USER**: After each AskUserQuestion, wait for the user's response before proceeding.

**Process:**
```
for each finding in sorted_findings (P1 first, then P2, then P3):
    1. Present THIS ONE finding with full details
    2. Use AskUserQuestion to ask what user wants to do
    3. WAIT for user response
    4. Execute user's choice (Apply/Skip/Stop)
    5. ONLY THEN move to next finding
```

---

**For each P1 finding (one at a time):**

Present the finding:
```
===============================================================
P1 - CRITICAL: <finding.title>
===============================================================

Agent: <finding.agent>
Category: <finding.category>
Location: <finding.location.file> code block #<finding.location.code_block>, line <finding.location.line>

Description:
  <finding.description>

Expected:
  <finding.expected>

Actual:
  <finding.actual>

Proposed Fix:
  <finding.proposed_fix>

```<language>
<finding.fix_code>
```

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Apply fix for <finding.id>: <finding.title>?"
- header: "Fix P1"
- options:
  - label: "Apply Fix"
    description: "Apply the proposed fix to <finding.location.file>"
  - label: "Apply with changes"
    description: "Provide custom instructions before applying"
  - label: "Skip"
    description: "Skip this fix, continue to next issue"
  - label: "Stop"
    description: "Stop fixing, show remaining issues"
```

**If "Apply with changes":**
1. Use AskUserQuestion to get user's modification instructions
2. Capture user's selection/input as `USER_MODIFICATIONS`
3. Continue to fix agent delegation below

**If "Apply Fix" or "Apply with changes":**

Delegate to fix-finding agent:

```
Use Task tool with:
- subagent_type: "architect:fix-finding"
- model: opus
- run_in_background: false  # Wait for result
- prompt: |
    Apply fix for OpenSpec verification finding.

    ## Finding Details
    - ID: <finding.id>
    - Severity: <finding.severity>
    - Category: <finding.category>
    - File: <finding.location.file>
    - Code Block: <finding.location.code_block>
    - Line: <finding.location.line>

    ## Problem
    <finding.description>

    ## Expected
    <finding.expected>

    ## Actual
    <finding.actual>

    ## Proposed Fix
    <finding.proposed_fix>

    ## Fix Code
    ```<language>
    <finding.fix_code>
    ```

    ## User Modifications (if any)
    <USER_MODIFICATIONS or "None - apply proposed fix as-is">

    ## Instructions
    1. Read the file (<finding.location.file>)
    2. Find code block #<finding.location.code_block>
    3. Locate the issue at approximately line <finding.location.line> within that code block
    4. Apply the fix (with user modifications if specified)
    5. Return JSON result with status

    Apply the fix and return JSON result.
```

Wait for agent completion, then parse the JSON result:

1. If status == "SUCCESS":
   ```
   ✓ Fixed <finding.id>: <finding.title>

   Continuing to next finding...
   ```
2. If status == "FAILED":
   ```
   Use AskUserQuestion with:
   - question: "Fix failed: <error>. What to do?"
   - header: "Fix Failed"
   - options:
     - label: "Skip"
       description: "Skip this fix and continue"
     - label: "Stop"
       description: "Stop fixing to investigate"
   ```
3. Continue to next finding

**If "Skip":**
- Mark finding as skipped
- Continue to next finding

**If "Stop":**
- Exit loop
- Show summary of remaining issues

**For P2 findings (one at a time):**

Same ONE BY ONE process as P1. Present each P2 finding individually, wait for user response, then proceed to next:
```
Use AskUserQuestion with:
- question: "Apply fix for <finding.id>: <finding.title>?"
- header: "Fix P2"
- options:
  - label: "Apply Fix"
    description: "Apply the proposed fix"
  - label: "Apply with changes"
    description: "Provide custom instructions before applying"
  - label: "Skip"
    description: "Skip this fix"
  - label: "Skip All P2/P3"
    description: "Skip remaining P2 and all P3 findings"
```

**For P3 findings:**

Show a summary first, then ask user preference:
```
===============================================================
P3 - NICE-TO-HAVE FINDINGS (<count>)
===============================================================

<id>: <title> (<file> block #<block>, line <line>)
<id>: <title> (<file> block #<block>, line <line>)
...

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "P3 findings are optional improvements. What to do?"
- header: "P3 Fixes"
- options:
  - label: "Skip all"
    description: "Skip all P3 findings and finish"
  - label: "Review each"
    description: "Go through P3 findings ONE BY ONE"
```

**If "Review each":**
Process P3 findings the same way as P1/P2 - present each finding individually, ask user what to do, wait for response, then proceed to next.

### Step 10: Final Report

```
===============================================================
OPENSPEC VERIFICATION COMPLETE: <CHANGE_NAME>
===============================================================

Results:
  Total Checks: <N>
  Passed: <X> / <N>

  Findings Addressed:
    Fixed: <A>
    Skipped: <B>
    Remaining: <C>

Fixes Applied:
  [x] <id>: <title> - FIXED
  [ ] <id>: <title> - SKIPPED
  ...

Remaining Issues (if any):
  P1: <count> critical issues
  P2: <count> important issues
  P3: <count> nice-to-have

Verification Status: <STATUS>

===============================================================
NEXT STEPS
===============================================================

<Based on status>

===============================================================
```

**Status determination:**
- `PASSED`: No findings, or all findings fixed
- `PASSED_WITH_WARNINGS`: Some P2/P3 remaining (proceed with caution)
- `HAS_ISSUES`: P1 issues remaining (warns but allows proceeding)

**Next steps by status:**

If PASSED:
```
OpenSpec verification passed!
Ready for implementation.

Suggested next steps:
1. Create beads: /architect:create-beads <SPEC_PATH>
2. Or implement directly: /architect:implement-beads
```

If PASSED_WITH_WARNINGS:
```
OpenSpec verified with warnings.
Consider addressing P2/P3 issues before implementation.

You may proceed, but review the warnings first.

Suggested next steps:
1. Review warnings above
2. Create beads: /architect:create-beads <SPEC_PATH>

To re-verify after fixes:
  /architect:verify-openspec <SPEC_PATH>
```

If HAS_ISSUES:
```
WARNING: Critical issues found in OpenSpec code.

These issues will likely cause problems during or after implementation.
Consider fixing them now to avoid rework later.

P1 Issues Remaining:
  - <id>: <title> (<file>)
  - <id>: <title> (<file>)

You may still proceed, but be aware of the risks.

To re-verify after fixes:
  /architect:verify-openspec <SPEC_PATH>
```

## Workflow Diagram

```
/architect:verify-openspec [spec-path]
    │
    ├─── [interactive] ─────────────────────────────────────────┐
    │    List available OpenSpec changes                        │
    │    AskUserQuestion: Select one                            │
    │                                                           │
    └─── [direct] ──────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    VALIDATE ENVIRONMENT                        │
│                                                               │
│  • Check OpenSpec installed/initialized                       │
│  • REQUIRE docs/architecture/INDEX.md                         │
│  • Validate spec path exists                                  │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                 READ OPENSPEC + EXTRACT CODE                   │
│                                                               │
│  • Read design.md → extract code blocks                       │
│  • Read tests.md → extract code blocks                        │
│  • Read proposal.md → affected files                          │
│  • Read architecture docs                                     │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│              LAUNCH 4 AGENTS IN PARALLEL                       │
│                                                               │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ │
│  │ typescript │ │architecture│ │  security  │ │performance │ │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘ │
│                                                               │
│  All agents receive code blocks + architecture docs           │
│  All run simultaneously with run_in_background: true          │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    WAIT FOR AGENTS                             │
│                                                               │
│  • TaskOutput for each agent (block: true)                    │
│  • Collect JSON findings from each                            │
│  • Handle timeouts/failures gracefully                        │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    CONSOLIDATE FINDINGS                        │
│                                                               │
│  • Merge from all agents                                      │
│  • Deduplicate same code_block/line/issue                    │
│  • Sort: P1 → P2 → P3                                        │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│              INTERACTIVE FIX LOOP (ONE BY ONE)                 │
│                                                               │
│  For EACH finding individually (P1 first, then P2, then P3): │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 1. Present ONE finding with full details                │  │
│  │ 2. AskUserQuestion: "Apply fix?"                        │  │
│  │ 3. WAIT for user response                               │  │
│  │ 4. Apply → Delegate to fix-finding agent                │  │
│  │    Skip → Mark as skipped                               │  │
│  │    Stop → Exit loop                                     │  │
│  │ 5. ONLY THEN proceed to next finding                    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  NEVER batch findings. ALWAYS one at a time.                  │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    FINAL REPORT                                │
│                                                               │
│  • Summary: fixed, skipped, remaining                         │
│  • Status: PASSED | PASSED_WITH_WARNINGS | HAS_ISSUES        │
│  • Next steps (warns but allows proceeding)                   │
└───────────────────────────────────────────────────────────────┘
```

## Agent Finding Format

All agents return findings in this JSON structure:

```json
{
  "agent": "verify-openspec-typescript",
  "summary": {
    "total_checks": 15,
    "passed": 12,
    "findings_count": 3
  },
  "findings": [
    {
      "id": "OSPEC-TS001",
      "severity": "P1",
      "category": "type_safety",
      "title": "Implicit any in function parameter",
      "description": "Parameter 'data' has implicit 'any' type",
      "location": {
        "file": "design.md",
        "code_block": 2,
        "line": 15
      },
      "expected": "Explicit type annotation",
      "actual": "function parse(data) { ... }",
      "proposed_fix": "Add type annotation",
      "fix_code": "function parse(data: ParseInput): ParseOutput { ... }"
    }
  ]
}
```

## Severity Classification

| Severity | Criteria | Recommendation |
|----------|----------|----------------|
| **P1 - Critical** | Security vulnerabilities, clear architecture violations | Fix before implementation |
| **P2 - Important** | SOLID violations, performance issues, anti-patterns | Recommended to fix |
| **P3 - Nice-to-Have** | Style, naming, optimization suggestions | Optional |

**Note:** Unlike verify-implementation, even P1 issues warn but allow proceeding. The user decides whether to fix or proceed.

## Git Policy

**NEVER push to git.** Do not run `git push` or any command that pushes to remote. The user will push manually when ready.

## Error Handling

| Scenario | Action |
|----------|--------|
| Architecture docs missing | **FAIL**: "Create docs/architecture/INDEX.md" |
| OpenSpec not installed | "Install OpenSpec: https://github.com/Fission-AI/OpenSpec" |
| OpenSpec not initialized | "Run: openspec init" |
| Spec path not found | Report error, suggest correct path |
| No design.md/tests.md | Report "No code to verify" |
| Agent times out | Report partial results, continue with others |
| Agent fails | Report error, continue with remaining agents |
| All agents fail | Report environment issue, suggest fixes |

## Example Usage

```bash
# Interactive mode (lists available changes)
/architect:verify-openspec

# Direct mode with specific spec
/architect:verify-openspec openspec/changes/add-auth/

# Typical workflow
/architect:create-openspec .claude/plans/add-auth-plan.md
/architect:verify-openspec openspec/changes/add-auth/
/architect:create-beads openspec/changes/add-auth/
/architect:implement-beads
```
