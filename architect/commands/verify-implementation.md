---
allowed-tools: Task, TaskOutput, Bash, Read, Glob, Grep, Edit, Write, AskUserQuestion
argument-hint: "[openspec-change-path]"
description: Post-implementation verification using parallel specialized agents
---

# Verify Implementation Command

Post-implementation verification that validates written code against OpenSpec specifications, design documents, and architecture patterns. Uses **6 specialized agents running in parallel** for comprehensive analysis.

**IMPORTANT**: Run this command AFTER `/architect:implement` completes to verify code quality, requirements consistency, and architecture compliance.

## Why Verify?

After implementation, code should be validated against:
- **TypeScript Quality**: Type safety, naming, testability
- **Architecture Compliance**: Patterns from docs/architecture/
- **Requirements Consistency**: Given/When/Then scenarios from specs/*.md
- **Exit Criteria**: Commands from tasks.md must pass
- **Security**: OWASP compliance, vulnerability detection
- **Performance**: N+1 queries, memory leaks, optimization

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

  Without architecture docs, the verify command cannot check compliance.
  ```

### Step 2: Determine Mode

**If $ARGUMENTS contains a path** (e.g., `openspec/changes/...`):
- Use direct mode - skip to Step 3

**If $ARGUMENTS is empty**:
- Use interactive mode - continue below

### Step 2a: List Available OpenSpec Changes (Interactive Mode)

```bash
# List completed or in-progress changes
ls -d openspec/changes/*/ 2>/dev/null | grep -v '/archive/' | while read dir; do
  change_name=$(basename "$dir")
  # Check if tasks.md has any completed tasks
  completed=$(grep -c '\[x\]' "$dir/tasks.md" 2>/dev/null || echo 0)
  total=$(grep -c '^\- \[' "$dir/tasks.md" 2>/dev/null || echo 0)
  echo "$change_name: $completed/$total tasks complete"
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
    description: "<N/M tasks complete>"
  - label: "<change-name-2>"
    description: "<N/M tasks complete>"
  ... (one option per available change, max 4)
```

**Handle Selection:**
- If user selects a change name → set `SPEC_PATH=openspec/changes/<selected>/`
- If user selects "Other" → use their custom input as path

**Edge Cases:**
- No changes found → "No OpenSpec changes found. Run `/create-openspec` first, then `/implement-beads`."
- No implemented changes → "No implemented changes found. Run `/implement-beads` first."

### Step 3: Read Spec Files

```bash
# Read all spec files
cat $SPEC_PATH/proposal.md
cat $SPEC_PATH/design.md
cat $SPEC_PATH/tasks.md
find $SPEC_PATH/specs -name "*.md" -exec cat {} \; 2>/dev/null
```

Store context:
- `CHANGE_NAME=$(basename $SPEC_PATH)`
- Parse `proposal.md` for affected files (Impact section)
- Parse `design.md` for Reference Implementation
- Parse `tasks.md` for Exit Criteria commands
- Parse `specs/*.md` for Given/When/Then scenarios

### Step 4: Launch 6 Verification Agents in Parallel

**CRITICAL**: Launch ALL agents simultaneously using Task tool with `run_in_background: true`.

Each agent receives:
- Change ID and spec path
- Full spec content (proposal.md, design.md, tasks.md, specs/*.md)
- List of affected files from proposal.md
- Architecture docs path: `docs/architecture/`

**Agent Launch Template:**

For each of the 6 agents, use:
```
Use Task tool with:
- subagent_type: "verify-<agent-name>"
- model: opus
- run_in_background: true
- prompt: |
    Verify implementation for OpenSpec change.

    Change ID: <CHANGE_NAME>
    Spec Path: <SPEC_PATH>

    ## OpenSpec Content

    ### proposal.md
    <full content>

    ### design.md
    <full content>

    ### tasks.md
    <full content>

    ### specs/*.md
    <full content of all spec files>

    ## Affected Files
    <list from proposal.md Impact section>

    ## Architecture Docs Path
    docs/architecture/

    Return findings as JSON in the standard format.
```

**Launch order** (all in parallel in a single message):
1. `verify-typescript` - TypeScript quality
2. `verify-architecture` - Architecture compliance
3. `verify-requirements` - Requirements consistency
4. `verify-exit-criteria` - Exit criteria execution
5. `verify-security` - Security vulnerabilities
6. `verify-performance` - Performance issues

Track agent IDs for later collection.

**CRITICAL: DO NOT EXIT after launching agents. You MUST proceed to Step 5 and wait for ALL agents to complete before continuing.**

### Step 5: Wait for All Agents

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

**DO NOT proceed to Step 6 until ALL agents have returned results. You MUST call TaskOutput for EACH agent and wait for completion.**

### Step 6: Consolidate and Sort Findings

After all agents complete:

1. **Merge findings** from all 6 agents into single list
2. **Deduplicate** findings with same file/line/issue
3. **Sort by severity**: P1 (Critical) → P2 (Important) → P3 (Nice-to-Have)
4. **Group by category** for presentation

**Calculate summary:**
```
total_findings = sum of all agent findings
p1_count = count where severity == "P1"
p2_count = count where severity == "P2"
p3_count = count where severity == "P3"
```

### Step 7: Present Results Summary

```
===============================================================
VERIFICATION RESULTS: <CHANGE_NAME>
===============================================================

Agents Completed: 6/6
Total Findings: <N> (P1: <X>, P2: <Y>, P3: <Z>)

By Agent:
  verify-typescript:     <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
  verify-architecture:   <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
  verify-requirements:   <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
  verify-exit-criteria:  <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
  verify-security:       <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
  verify-performance:    <N> findings (P1: <X>, P2: <Y>, P3: <Z>)

===============================================================
```

**If no findings**: Skip to Step 9 (Final Report) with PASSED status.

**If findings exist**: Proceed IMMEDIATELY to Step 8. Do NOT ask the user if they want to fix all findings at once. Do NOT present a batch fix option. Show findings ONE AT A TIME.

### Step 8: Interactive Fix Loop

Process findings in severity order: P1 first, then P2, then P3.

**For each P1 finding:**

Present the finding:
```
===============================================================
P1 - CRITICAL: <finding.title>
===============================================================

Agent: <finding.agent>
Category: <finding.category>
File: <finding.location.file>:<finding.location.line>

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
1. Use AskUserQuestion to get user's modification instructions:
   ```
   Use AskUserQuestion with:
   - question: "What changes or additions do you want to make to the proposed fix?"
   - header: "Modify Fix"
   - options:
     - label: "Add logging"
       description: "Add console.log or logger statements"
     - label: "Add error handling"
       description: "Wrap in try-catch or add validation"
     - label: "Different approach"
       description: "Describe your preferred solution"
   ```
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
    Apply fix for verification finding.

    ## Finding Details
    - ID: <finding.id>
    - Severity: <finding.severity>
    - Category: <finding.category>
    - File: <finding.location.file>
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

    Apply the fix and run verification. Return JSON result.
```

Wait for agent completion, then parse the JSON result:

1. If status == "SUCCESS":
   ```
   ✓ Fixed <finding.id>: <finding.title>

   Verification: PASSED

   Continuing to next finding...
   ```
2. If status == "FAILED" or status == "PARTIAL":
   ```
   Use AskUserQuestion with:
   - question: "Fix applied but verification failed: <verification_output>. What to do?"
   - header: "Fix Failed"
   - options:
     - label: "Keep fix anyway"
       description: "Keep the applied fix and continue"
     - label: "Revert fix"
       description: "Undo the fix and skip this issue"
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

**For P2 findings:**

Same process as P1 (including agent delegation), but with these options:
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

Show summary instead of individual fixes:
```
===============================================================
P3 - NICE-TO-HAVE FINDINGS (<count>)
===============================================================

<id>: <title> (<file>:<line>)
<id>: <title> (<file>:<line>)
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
    description: "Go through P3 findings one by one"
```

### Step 9: Final Report

```
===============================================================
VERIFICATION COMPLETE: <CHANGE_NAME>
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
- `PASSED_WITH_WARNINGS`: No P1 remaining, some P2/P3 remaining
- `FAILED`: P1 issues remaining

**Next steps by status:**

If PASSED:
```
Implementation verified successfully!
Ready for review/merge.

Suggested next steps:
1. Review changes: git diff
2. Commit: git add . && git commit -m "feat: <change description>"
3. Create PR or merge
```

If PASSED_WITH_WARNINGS:
```
Implementation verified with warnings.
Consider addressing P2/P3 issues before merge.

To re-verify after fixes:
  /architect:verify-implementation <SPEC_PATH>
```

If FAILED:
```
Critical issues remain. Address P1 issues before merge.

P1 Issues Remaining:
  - <id>: <title> (<file>)
  - <id>: <title> (<file>)

To re-verify after fixes:
  /architect:verify-implementation <SPEC_PATH>
```

## Workflow Diagram

```
/architect:verify-implementation [spec-path]
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
│                    READ SPEC FILES                             │
│                                                               │
│  • proposal.md, design.md, tasks.md                           │
│  • specs/*.md (all spec files)                                │
│  • Extract affected files, exit criteria, scenarios           │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│              LAUNCH 6 AGENTS IN PARALLEL                       │
│                                                               │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                │
│  │ typescript │ │architecture│ │requirements│                │
│  └────────────┘ └────────────┘ └────────────┘                │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                │
│  │exit-criteri│ │  security  │ │performance │                │
│  └────────────┘ └────────────┘ └────────────┘                │
│                                                               │
│  All agents run simultaneously with run_in_background: true   │
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
│  • Deduplicate same file/line/issue                          │
│  • Sort: P1 → P2 → P3                                        │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    INTERACTIVE FIX LOOP                        │
│                                                               │
│  For each finding (P1 first, then P2, then P3):              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Present finding with proposed fix                       │  │
│  │ Ask: "Apply fix?"                                       │  │
│  │ Apply → Edit file → Re-verify → Report                 │  │
│  │ Skip → Continue to next                                 │  │
│  │ Stop → Exit loop                                        │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    FINAL REPORT                                │
│                                                               │
│  • Summary: fixed, skipped, remaining                         │
│  • Status: PASSED | PASSED_WITH_WARNINGS | FAILED            │
│  • Next steps based on status                                 │
└───────────────────────────────────────────────────────────────┘
```

## Agent Finding Format

All agents return findings in this JSON structure:

```json
{
  "agent": "verify-typescript",
  "summary": {
    "total_checks": 15,
    "passed": 12,
    "findings_count": 3
  },
  "findings": [
    {
      "id": "TS001",
      "severity": "P1",
      "category": "type_safety",
      "title": "Implicit any in function parameter",
      "description": "Parameter 'data' has implicit 'any' type",
      "location": { "file": "src/utils/parser.ts", "line": 45 },
      "expected": "Explicit type annotation",
      "actual": "function parse(data) { ... }",
      "proposed_fix": "Add type annotation",
      "fix_code": "function parse(data: ParseInput): ParseOutput { ... }"
    }
  ]
}
```

## Severity Classification

| Severity | Criteria | Blocks Merge? |
|----------|----------|---------------|
| **P1 - Critical** | Breaks functionality, security vulnerability, fails exit criteria | Yes |
| **P2 - Important** | Should fix, architecture violation, code quality issue | Recommended |
| **P3 - Nice-to-Have** | Optimization, style, manual verification reminder | No |

## Git Policy

**NEVER push to git.** Do not run `git push` or any command that pushes to remote. The user will push manually when ready.

## Error Handling

| Scenario | Action |
|----------|--------|
| Architecture docs missing | **FAIL**: "Create docs/architecture/INDEX.md" |
| OpenSpec not installed | "Install OpenSpec: https://github.com/Fission-AI/OpenSpec" |
| OpenSpec not initialized | "Run: openspec init" |
| Spec path not found | Report error, suggest correct path |
| Agent times out | Report partial results, continue with others |
| Agent fails | Report error, continue with remaining agents |
| All agents fail | Report environment issue, suggest fixes |

## Example Usage

```bash
# Interactive mode (lists available changes)
/architect:verify-implementation

# Direct mode with specific spec
/architect:verify-implementation openspec/changes/add-auth/

# After verification completes successfully
git add . && git commit -m "feat: add authentication"
```
