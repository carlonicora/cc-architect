---
allowed-tools: Bash, Read, Write, Edit, Glob, AskUserQuestion
argument-hint: "[spec-path]"
description: Interactive sprint-meeting-style review of OpenSpec proposals with editing
---

# OpenSpec Review

Interactive, sprint-meeting-style review of OpenSpec proposals. The AI acts as a project manager, walking through specifications segment-by-segment, validating business logic, architecture, and development details. User feedback triggers immediate edits to the OpenSpec files with cascading updates.

## Why Review?

Before converting an OpenSpec to beads and implementing, it's critical to validate:
- **Business Logic**: Are we solving the right problem?
- **Architecture**: Is the technical approach sound?
- **Development**: Are implementation details correct?

This command provides a structured walkthrough that catches issues before code is written.

## Arguments

- **No argument**: Interactive mode - lists available OpenSpec changes and asks which to review
- **Spec path**: Direct mode - uses the provided path (e.g., `openspec/changes/add-auth/`)

## Instructions

### Step 1: Validate Environment

```bash
# Check openspec is installed
openspec version 2>&1 | head -1

# Check openspec is initialized
ls openspec/project.md 2>/dev/null && echo "OpenSpec initialized" || echo "Not initialized"
```

If not installed: "Install OpenSpec: https://github.com/Fission-AI/OpenSpec"
If not initialized: "Run: openspec init"

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
  # Get brief description from proposal.md
  desc=$(grep -A1 "^# Change:" "$dir/proposal.md" 2>/dev/null | tail -1 | head -c 60)
  [ -z "$desc" ] && desc=$(grep -m1 "^## Why" -A1 "$dir/proposal.md" 2>/dev/null | tail -1 | head -c 60)
  echo "$change_name: $desc"
done
```

**Present Available Changes:**

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Select an OpenSpec change to review:"
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

### Step 3: Read and Parse All Spec Files

```bash
# Read all spec files
cat $SPEC_PATH/proposal.md
cat $SPEC_PATH/design.md
cat $SPEC_PATH/tasks.md
find $SPEC_PATH/specs -name "*.md" -exec cat {} \; 2>/dev/null
```

Parse and identify:
- **Proposal sections**: Why, What Changes, Impact, Success Criteria, Out of Scope
- **Requirements**: All `### Requirement:` blocks from specs/*.md
- **Design sections**: Decisions, Reference Implementation, Migration Patterns, Risk Analysis
- **Tasks sections**: Phases, Exit Criteria, Success Conditions

Store the change name for reporting: `CHANGE_NAME=$(basename $SPEC_PATH)`

### Step 4: Review Proposal.md (Section by Section)

For each section in proposal.md, conduct a focused review.

#### 4.1: Review "Why" (Motivation)

Present the section:
```
===============================================================
REVIEWING: Why (Motivation)
===============================================================

<content of "## Why" section>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Is the motivation for this change correct? Does it solve the right problem?"
- header: "Why"
- options:
  - label: "Yes, continue"
    description: "The motivation is correct, move to the next section"
  - label: "No, needs changes"
    description: "I have feedback or corrections for this section"
```

**If "Yes, continue"**: Move to 4.2

**If "No, needs changes"**:
1. Say: "What changes are needed for the motivation?"
2. Wait for user's response
3. Edit proposal.md immediately with the feedback
4. Show the edit: "Updated the 'Why' section in proposal.md with: <summary of change>"
5. Ask confirmation:
```
Use AskUserQuestion with:
- question: "Is this update correct?"
- header: "Confirm"
- options:
  - label: "Yes, continue"
    description: "Update is correct, move to next section"
  - label: "No, adjust further"
    description: "Make additional adjustments"
```
6. If "No, adjust further" → repeat from step 1
7. Continue to 4.2

#### 4.2: Review "What Changes" (Scope)

Present the section:
```
===============================================================
REVIEWING: What Changes (Scope)
===============================================================

<content of "## What Changes" section>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Is the scope of changes correct? Are all changes listed?"
- header: "Scope"
- options:
  - label: "Yes, continue"
    description: "The scope is correct, move to the next section"
  - label: "No, needs changes"
    description: "I have feedback or corrections for the scope"
```

Handle "No" the same way as 4.1 - open discussion, edit, confirm.

#### 4.3: Review "Impact" (Affected Code/Specs)

Present the section:
```
===============================================================
REVIEWING: Impact (Affected Code/Specs)
===============================================================

<content of "## Impact" section>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Is the impact assessment complete? Are all affected areas identified?"
- header: "Impact"
- options:
  - label: "Yes, continue"
    description: "Impact assessment is complete, move to the next section"
  - label: "No, needs changes"
    description: "I have feedback or corrections for the impact"
```

Handle "No" the same way - open discussion, edit, confirm.

#### 4.4: Review "Success Criteria"

Present the section:
```
===============================================================
REVIEWING: Success Criteria
===============================================================

<content of "## Success Criteria" section>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Are the success criteria measurable and complete?"
- header: "Criteria"
- options:
  - label: "Yes, continue"
    description: "Success criteria are correct, move to the next section"
  - label: "No, needs changes"
    description: "I have feedback or corrections for success criteria"
```

Handle "No" the same way - open discussion, edit, confirm.

**Cascading**: If success criteria change, also check and update tasks.md "Success Conditions" section.

#### 4.5: Review "Out of Scope"

Present the section:
```
===============================================================
REVIEWING: Out of Scope
===============================================================

<content of "## Out of Scope" section>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Is the out-of-scope section clear? Are boundaries well-defined?"
- header: "Boundaries"
- options:
  - label: "Yes, continue"
    description: "Boundaries are clear, proceed to requirements review"
  - label: "No, needs changes"
    description: "I have feedback or corrections for boundaries"
```

Handle "No" the same way - open discussion, edit, confirm.

### Step 5: Review Requirements (specs/*.md)

For each requirement in specs/*.md, conduct a 3-level validation.

**List all requirements first:**
```bash
grep -h "^### Requirement:" $SPEC_PATH/specs/*/spec.md 2>/dev/null
```

Report: "Found N requirements to review."

**For each requirement:**

#### 5.1: Business Logic Validation

Present the requirement:
```
===============================================================
REQUIREMENT: <Requirement Name>
Level: BUSINESS LOGIC
===============================================================

<Full requirement text including all scenarios>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Is this business requirement correct? Does it solve the right problem?"
- header: "Business"
- options:
  - label: "Yes, continue"
    description: "Business logic is correct, proceed to architecture review"
  - label: "No, needs changes"
    description: "I have feedback on the business requirement"
```

**If "No, needs changes"**:
1. Say: "What changes are needed for this business requirement?"
2. Wait for user's response
3. Edit the spec file immediately
4. **Cascading**: Check if related tasks need updating in tasks.md
5. Show changes: "Updated requirement in specs/<area>/spec.md. Also updated related tasks in tasks.md."
6. Confirm the edit, then continue

#### 5.2: Architecture Validation

Find related design decisions from design.md based on the requirement name/area.

Present:
```
===============================================================
REQUIREMENT: <Requirement Name>
Level: ARCHITECTURE
===============================================================

## Related Design Decisions

<Relevant decisions from design.md that relate to this requirement>

## Component Design

<Relevant component/interface definitions>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Is the technical approach for this requirement sound?"
- header: "Architecture"
- options:
  - label: "Yes, continue"
    description: "Architecture is sound, proceed to development review"
  - label: "No, needs changes"
    description: "I have feedback on the technical approach"
```

**If "No, needs changes"**:
1. Say: "What changes are needed for the technical approach?"
2. Wait for user's response
3. Edit design.md immediately
4. **Cascading**: Check if related Reference Implementation code needs updating
5. Show changes and confirm

#### 5.3: Development Validation

Find related tasks from tasks.md based on the requirement name/area.

Present:
```
===============================================================
REQUIREMENT: <Requirement Name>
Level: DEVELOPMENT
===============================================================

## Related Tasks

<Tasks from tasks.md that implement this requirement>

## Exit Criteria for This Requirement

<Relevant exit criteria commands>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Are the implementation details correct for this requirement?"
- header: "Development"
- options:
  - label: "Yes, continue"
    description: "Implementation details are correct, proceed to next requirement"
  - label: "No, needs changes"
    description: "I have feedback on implementation details"
```

**If "No, needs changes"**:
1. Say: "What changes are needed for the implementation details?"
2. Wait for user's response
3. Edit tasks.md immediately
4. Show changes and confirm

**After completing all 3 levels, move to the next requirement.**

### Step 6: Review Design.md (Remaining Sections)

Review sections not covered during requirements review:

#### 6.1: Reference Implementation

If design.md has a "Reference Implementation" section with code:

Present:
```
===============================================================
REVIEWING: Reference Implementation
===============================================================

<Summary of files and code in Reference Implementation section>

Files:
- <file1>: <line count> lines
- <file2>: <line count> lines
...

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Is the reference implementation code complete and accurate?"
- header: "Code"
- options:
  - label: "Yes, continue"
    description: "Reference implementation is correct"
  - label: "No, needs changes"
    description: "I have feedback on the reference implementation"
```

Handle "No" with discussion → edit → confirm.

#### 6.2: Migration Patterns

If design.md has migration patterns:

Present:
```
===============================================================
REVIEWING: Migration Patterns
===============================================================

<Summary of before/after patterns>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Are the migration patterns (before/after) correct?"
- header: "Migration"
- options:
  - label: "Yes, continue"
    description: "Migration patterns are correct"
  - label: "No, needs changes"
    description: "I have feedback on migration patterns"
```

Handle "No" with discussion → edit → confirm.

#### 6.3: Risk Analysis

If design.md has risk analysis:

Present:
```
===============================================================
REVIEWING: Risk Analysis
===============================================================

## Technical Risks
<risk table>

## Integration Risks
<risk table>

## Rollback Strategy
<rollback details>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Is the risk analysis complete? Are mitigations adequate?"
- header: "Risks"
- options:
  - label: "Yes, continue"
    description: "Risk analysis is adequate"
  - label: "No, needs changes"
    description: "I have feedback on risk analysis"
```

Handle "No" with discussion → edit → confirm.

### Step 7: Review Tasks.md (Final Validation)

#### 7.1: Task Phases and Ordering

Present:
```
===============================================================
REVIEWING: Task Phases
===============================================================

<List of phases with task counts>

Phase 1: <name> - N tasks
Phase 2: <name> - N tasks
...
Phase N: Validation - N tasks

Total Tasks: <count>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Is the task ordering and phasing correct?"
- header: "Phases"
- options:
  - label: "Yes, continue"
    description: "Task phases are well-organized"
  - label: "No, needs changes"
    description: "I have feedback on task organization"
```

Handle "No" with discussion → edit → confirm.

#### 7.2: Exit Criteria

Present:
```
===============================================================
REVIEWING: Exit Criteria
===============================================================

Commands to run:
<exact commands from Exit Criteria section>

Success Conditions:
<list from Success Conditions section>

===============================================================
```

Use AskUserQuestion:
```
Use AskUserQuestion with:
- question: "Are the exit criteria and success conditions complete?"
- header: "Exit Criteria"
- options:
  - label: "Yes, complete review"
    description: "Exit criteria are complete, finish the review"
  - label: "No, needs changes"
    description: "I have feedback on exit criteria"
```

Handle "No" with discussion → edit → confirm.

### Step 8: Completion

```
===============================================================
OPENSPEC REVIEW COMPLETE
===============================================================

Change: <CHANGE_NAME>
Path: <SPEC_PATH>

Sections Reviewed:
  Proposal.md:
    [x] Why (Motivation)
    [x] What Changes (Scope)
    [x] Impact
    [x] Success Criteria
    [x] Out of Scope

  Requirements (specs/*.md):
    [x] <requirement-1> (Business + Architecture + Development)
    [x] <requirement-2> (Business + Architecture + Development)
    ...

  Design.md:
    [x] Reference Implementation
    [x] Migration Patterns
    [x] Risk Analysis

  Tasks.md:
    [x] Task Phases
    [x] Exit Criteria

Edits Made: <count> (if any)

===============================================================
NEXT STEPS
===============================================================

The OpenSpec has been validated and is ready for implementation.

1. Convert to beads: /beads <SPEC_PATH>
2. Then implement: /implement

===============================================================
```

## Workflow Diagram

```
/architect:review [spec-path]
    │
    ├─── [interactive] ─────────────────────────────────────────┐
    │    List available OpenSpec changes                        │
    │    AskUserQuestion: Select one                            │
    │                                                           │
    └─── [direct] ──────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    PROPOSAL.MD REVIEW                         │
│                                                               │
│  For each section:                                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Present section                                         │  │
│  │ Ask: "Is this correct?"                                 │  │
│  │ Yes → next section                                      │  │
│  │ No → Discussion → Edit → Confirm → next section         │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  Sections: Why → What Changes → Impact → Criteria → Scope    │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    REQUIREMENTS REVIEW                        │
│                                                               │
│  For each requirement:                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ BUSINESS: "Is this requirement correct?"                │  │
│  │ Yes → Architecture | No → Edit spec + tasks → confirm   │  │
│  ├─────────────────────────────────────────────────────────┤  │
│  │ ARCHITECTURE: "Is the approach sound?"                  │  │
│  │ Yes → Development | No → Edit design → confirm          │  │
│  ├─────────────────────────────────────────────────────────┤  │
│  │ DEVELOPMENT: "Are implementation details correct?"      │  │
│  │ Yes → next req | No → Edit tasks → confirm              │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    DESIGN.MD REVIEW                           │
│                                                               │
│  Reference Implementation → Migration Patterns → Risk Analysis│
│  Same Yes/No pattern with immediate edits                     │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    TASKS.MD REVIEW                            │
│                                                               │
│  Task Phases → Exit Criteria                                  │
│  Same Yes/No pattern with immediate edits                     │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    COMPLETION                                 │
│                                                               │
│  Report: "Review complete"                                    │
│  Suggest: "/beads <path>" → "/implement"                      │
└───────────────────────────────────────────────────────────────┘
```

## Cascading Updates

When editing one file, related files may need updates:

| Edit | Also Update |
|------|-------------|
| Requirement in specs/*.md | Related tasks in tasks.md |
| Success criteria in proposal.md | Success conditions in tasks.md |
| Design decision in design.md | Reference implementation code in design.md |
| Task scope in tasks.md | Exit criteria commands |

Always inform the user: "I've also updated <file> to reflect this change."

## Edit Flow Pattern

When user selects "No, needs changes":

```
1. AI: "What changes are needed for this section?"
2. User: <provides feedback>
3. AI: Edits the file immediately using Edit or Write tool
4. AI: Shows summary: "Updated <file> with: <brief summary>"
5. AI: Checks for cascading updates needed
6. AI: If cascading needed: "I've also updated <other-file> to keep consistency."
7. AI: Asks confirmation:
   AskUserQuestion:
   - question: "Is this update correct?"
   - header: "Confirm"
   - options:
     - label: "Yes, continue"
     - label: "No, adjust further"
8. If "No, adjust further": repeat from step 1
9. Continue to next section
```

## Git Policy

**NEVER push to git.** Do not run `git push` or any command that pushes to remote. The user will push manually when ready.

## Error Handling

| Scenario | Action |
|----------|--------|
| OpenSpec not installed | "Install OpenSpec: https://github.com/Fission-AI/OpenSpec" |
| OpenSpec not initialized | "Run: openspec init" |
| No changes found | "No OpenSpec changes found. Run `/openspec` first." |
| Path not found | Report error, suggest correct path |
| Spec file missing | Report which file is missing, cannot proceed |

## Example Usage

```bash
# Interactive mode (lists available changes)
/architect:review

# Direct mode with specific spec
/architect:review openspec/changes/add-auth/

# After review completes
/beads openspec/changes/add-auth/
/implement
```
