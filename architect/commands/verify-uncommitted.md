---
allowed-tools: Task, TaskOutput, Bash, Read, Glob, Grep, Edit, Write, AskUserQuestion
argument-hint: "[--staged | --all | path...]"
description: Verify uncommitted git changes using parallel verification agents (no OpenSpec required)
---

# Verify Uncommitted Command

Verification command that validates uncommitted git changes using **4 specialized agents running in parallel**. Unlike `/architect:verify`, this command does NOT require OpenSpec - it works directly with git-tracked changes.

**Use this command** for:
- Quick verification before committing
- Code review assistance (`--staged`)
- Ad-hoc verification of specific files

## What Gets Checked

| Agent | Checks | Works Without OpenSpec? |
|-------|--------|------------------------|
| **verify-typescript** | Type safety, naming, testability | Yes (full) |
| **verify-security** | OWASP compliance, vulnerabilities | Yes (full) |
| **verify-performance** | N+1 queries, memory leaks, optimization | Yes (full) |
| **verify-architecture** | SOLID, boundaries, patterns | Yes (partial - skips design.md comparison) |

**NOT included** (require OpenSpec):
- verify-requirements (needs Given/When/Then scenarios)
- verify-exit-criteria (needs tasks.md commands)

## Arguments

| Argument | Behavior |
|----------|----------|
| (no args) | Verify all uncommitted files (staged + unstaged + untracked) |
| `--staged` | Only verify staged files |
| `--all` | Explicit: all uncommitted (same as no args) |
| `path...` | Verify specific files (space-separated) |

## Instructions

### Step 1: Validate Environment

```bash
# Check we're in a git repository
git rev-parse --is-inside-work-tree 2>/dev/null && echo "Git repository" || echo "NOT_A_GIT_REPO"

# Check architecture docs (optional but recommended)
ls docs/architecture/INDEX.md 2>/dev/null && echo "Architecture docs found" || echo "ARCH_DOCS_MISSING"
```

**Error Handling:**
- If NOT a git repo: **FAIL** with error:
  ```
  Not a git repository.

  Initialize with: git init

  Or run this command from within an existing git repository.
  ```

- If architecture docs missing: **WARNING** (continue):
  ```
  WARNING: docs/architecture/INDEX.md not found.
  Architecture checks will be limited to SOLID principles and general patterns.
  For full architecture verification, create docs/architecture/ with your project patterns.
  ```

### Step 2: Determine Mode and Get Files

**Parse $ARGUMENTS:**

**If `--staged`:**
```bash
# Get only staged TypeScript/JavaScript files
git diff --cached --name-only | grep -E '\.(ts|tsx|js|jsx)$' || true
```

**If specific paths provided** (not starting with `--`):
```bash
# Use the provided paths, filter to existing files
for path in $ARGUMENTS; do
  if [[ -f "$path" ]] && [[ "$path" =~ \.(ts|tsx|js|jsx)$ ]]; then
    echo "$path"
  fi
done
```

**If `--all` or no arguments (default):**
```bash
# Get all uncommitted TypeScript/JavaScript files (staged + unstaged + untracked)
{ git diff --cached --name-only; git diff --name-only; git ls-files --others --exclude-standard; } | sort -u | grep -E '\.(ts|tsx|js|jsx)$' || true
```

Store result as `FILES_TO_VERIFY`.

**If no files found:**
```
No uncommitted TypeScript/JavaScript files found. Nothing to verify.

To verify specific files:
  /architect:verify-uncommitted path/to/file.ts

To verify after making changes:
  1. Make code changes
  2. Run: /architect:verify-uncommitted
```
**Exit early** - do not proceed to agent launch.

### Step 3: Display File Summary

```
===============================================================
VERIFY UNCOMMITTED CHANGES
===============================================================

Mode: <staged | all uncommitted | specific files>
Files to verify: <count> TypeScript/JavaScript files

<list each file, one per line>

Launching 4 verification agents...
===============================================================
```

### Step 4: Launch 4 Verification Agents in Parallel

**CRITICAL**: Launch ALL agents simultaneously using Task tool with `run_in_background: true`.

**Agent Launch Template:**

For each of the 4 agents, use:
```
Use Task tool with:
- subagent_type: "architect:verify-<agent-name>"
- model: opus
- run_in_background: true
- prompt: <agent-specific prompt below>
```

**verify-typescript prompt:**
```
Verify uncommitted git changes for TypeScript quality (no OpenSpec context).

## Mode
Git uncommitted file verification - no OpenSpec available.
Skip any checks that require OpenSpec content.

## Files to Verify
<FILES_TO_VERIFY - one per line>

## Architecture Docs Path
docs/architecture/

## Instructions
1. Focus ONLY on the files listed above
2. Do not expect or reference OpenSpec content
3. Run TypeScript compiler, check for any usage, naming, etc.
4. Return findings as JSON in the standard format
```

**verify-security prompt:**
```
Verify uncommitted git changes for security vulnerabilities (no OpenSpec context).

## Mode
Git uncommitted file verification - no OpenSpec available.
Skip any checks that require OpenSpec content.

## Files to Verify
<FILES_TO_VERIFY - one per line>

## Instructions
1. Focus ONLY on the files listed above
2. Do not expect or reference OpenSpec content
3. Check OWASP Top 10, hardcoded secrets, injection, XSS, etc.
4. Return findings as JSON in the standard format
```

**verify-performance prompt:**
```
Verify uncommitted git changes for performance issues (no OpenSpec context).

## Mode
Git uncommitted file verification - no OpenSpec available.
Skip any checks that require OpenSpec content.

## Files to Verify
<FILES_TO_VERIFY - one per line>

## Instructions
1. Focus ONLY on the files listed above
2. Do not expect or reference OpenSpec content
3. Check N+1 patterns, memory leaks, missing memoization, etc.
4. Return findings as JSON in the standard format
```

**verify-architecture prompt:**
```
Verify uncommitted git changes for architecture compliance (no OpenSpec context).

## Mode
Git uncommitted file verification - no OpenSpec available.
Skip any checks that require OpenSpec content.

## Files to Verify
<FILES_TO_VERIFY - one per line>

## Architecture Docs Path
docs/architecture/

## IMPORTANT: Modified Scope
SKIP Phase 2 (Design.md Compliance) - no design.md available.

Focus ONLY on:
- SOLID principles (SRP, OCP, LSP, ISP, DIP)
- Layer boundary violations
- Component boundary violations
- Circular dependencies
- Pattern consistency with docs/architecture/
- Anti-patterns from docs/architecture/anti-patterns.md
- File organization

Do NOT check:
- Design.md compliance (no design.md available)
- Reference implementation comparison (no reference available)

## Instructions
1. Focus ONLY on the files listed above
2. Read docs/architecture/ if available for pattern reference
3. Return findings as JSON in the standard format
```

**Launch order** (all in parallel in a single message):
1. `architect:verify-typescript` - TypeScript quality
2. `architect:verify-architecture` - Architecture compliance (partial)
3. `architect:verify-security` - Security vulnerabilities
4. `architect:verify-performance` - Performance issues

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
- If agent times out: report partial results, note timeout
- If agent fails: report error, continue with others
- If all agents fail: report environment issue, suggest fixes

**DO NOT proceed to Step 6 until ALL agents have returned results. You MUST call TaskOutput for EACH agent and wait for completion.**

### Step 6: Consolidate and Sort Findings

After all agents complete:

1. **Merge findings** from all 4 agents into single list
2. **Deduplicate** findings with same file/line/issue
3. **Sort by severity**: P1 (Critical) -> P2 (Important) -> P3 (Nice-to-Have)
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
VERIFICATION RESULTS: Uncommitted Changes
===============================================================

Files Verified: <N> TypeScript/JavaScript files
Agents Completed: <N>/4

Total Findings: <N> (P1: <X>, P2: <Y>, P3: <Z>)

By Agent:
  verify-typescript:     <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
  verify-architecture:   <N> findings (P1: <X>, P2: <Y>, P3: <Z>)
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
VERIFICATION COMPLETE: Uncommitted Changes
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
Verification passed!
Ready for commit.

Suggested next steps:
1. Review changes: git diff
2. Commit: git add . && git commit -m "<commit message>"
```

If PASSED_WITH_WARNINGS:
```
Verification passed with warnings.
Consider addressing P2/P3 issues before commit.

To re-verify after fixes:
  /architect:verify-uncommitted
```

If FAILED:
```
Critical issues remain. Address P1 issues before commit.

P1 Issues Remaining:
  - <id>: <title> (<file>)
  - <id>: <title> (<file>)

To re-verify after fixes:
  /architect:verify-uncommitted
```

## Workflow Diagram

```
/architect:verify-uncommitted [args]
    |
    +--- [--staged] --> git diff --cached --name-only
    |
    +--- [--all/default] --> staged + unstaged + untracked
    |
    +--- [paths...] --> use provided paths
                                    |
                                    v
+---------------------------------------------------------------+
|                    VALIDATE ENVIRONMENT                        |
|                                                               |
|  * Check git repository (REQUIRED)                            |
|  * Check docs/architecture/ (optional, warn if missing)       |
+---------------------------------------------------------------+
                                    |
                                    v
+---------------------------------------------------------------+
|                    GET FILES FROM GIT                          |
|                                                               |
|  * Filter to TypeScript/JavaScript files                      |
|  * Exit early if no files found                               |
+---------------------------------------------------------------+
                                    |
                                    v
+---------------------------------------------------------------+
|              LAUNCH 4 AGENTS IN PARALLEL                       |
|                                                               |
|  +------------+ +------------+ +------------+ +------------+  |
|  | typescript | |architecture| |  security  | |performance |  |
|  +------------+ +------------+ +------------+ +------------+  |
|                                                               |
|  All agents run simultaneously with run_in_background: true   |
+---------------------------------------------------------------+
                                    |
                                    v
+---------------------------------------------------------------+
|                    WAIT FOR AGENTS                             |
|                                                               |
|  * TaskOutput for each agent (block: true)                    |
|  * Collect JSON findings from each                            |
|  * Handle timeouts/failures gracefully                        |
+---------------------------------------------------------------+
                                    |
                                    v
+---------------------------------------------------------------+
|                    CONSOLIDATE FINDINGS                        |
|                                                               |
|  * Merge from all agents                                      |
|  * Deduplicate same file/line/issue                          |
|  * Sort: P1 -> P2 -> P3                                      |
+---------------------------------------------------------------+
                                    |
                                    v
+---------------------------------------------------------------+
|                    INTERACTIVE FIX LOOP                        |
|                                                               |
|  For each finding (P1 first, then P2, then P3):              |
|  +----------------------------------------------------------+ |
|  | Present finding with proposed fix                        | |
|  | Ask: "Apply fix?"                                        | |
|  | Apply -> Edit file -> Re-verify -> Report               | |
|  | Skip -> Continue to next                                 | |
|  | Stop -> Exit loop                                        | |
|  +----------------------------------------------------------+ |
+---------------------------------------------------------------+
                                    |
                                    v
+---------------------------------------------------------------+
|                    FINAL REPORT                                |
|                                                               |
|  * Summary: fixed, skipped, remaining                         |
|  * Status: PASSED | PASSED_WITH_WARNINGS | FAILED            |
|  * Next steps based on status                                 |
+---------------------------------------------------------------+
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

| Severity | Criteria | Blocks Commit? |
|----------|----------|----------------|
| **P1 - Critical** | Type errors, security vulnerabilities, circular imports | Yes |
| **P2 - Important** | SOLID violations, missing validation, performance issues | Recommended |
| **P3 - Nice-to-Have** | Naming, style, optimization suggestions | No |

## Comparison with /architect:verify

| Feature | `/architect:verify` | `/architect:verify-uncommitted` |
|---------|---------------------|-------------------------------|
| Requires OpenSpec | Yes | No |
| Agents | 6 | 4 |
| File source | OpenSpec proposal.md | Git uncommitted |
| Architecture docs | Required | Optional |
| Use case | After /implement | Before commit |

## Git Policy

**NEVER push to git.** Do not run `git push` or any command that pushes to remote. The user will push manually when ready.

## Error Handling

| Scenario | Action |
|----------|--------|
| Not a git repository | **FAIL**: "Not a git repository. Initialize with: git init" |
| No uncommitted files | "No uncommitted TypeScript files found. Nothing to verify." (exit early) |
| Architecture docs missing | **WARNING**: "docs/architecture/INDEX.md not found. Architecture checks will be limited." (continue) |
| Agent times out | Report partial results, continue with others |
| Agent fails | Report error, continue with remaining agents |
| All agents fail | Report environment issue, suggest fixes |

## Example Usage

```bash
# Verify all uncommitted changes (default)
/architect:verify-uncommitted

# Verify only staged files (code review mode)
/architect:verify-uncommitted --staged

# Verify specific files
/architect:verify-uncommitted src/utils/parser.ts src/components/Button.tsx

# After verification passes
git add . && git commit -m "feat: add parser utility"
```
