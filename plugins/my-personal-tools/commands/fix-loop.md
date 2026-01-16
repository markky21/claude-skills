---
description: Iteratively validate and fix code issues until no critical/high findings remain
allowed-tools: Bash(git *), Task, AskUserQuestion, TodoWrite, Read, Edit, Glob, Grep
---

# Fix Loop Command

Iteratively validates code and fixes issues until no critical/high severity findings remain (inspired by the [Ralph Wiggum technique](https://awesomeclaude.ai/ralph-wiggum)).

## Context

- Current branch: !`git branch --show-current`
- Base branch: !`git merge-base HEAD main 2>/dev/null && echo "main" || echo "HEAD~5"`
- Files changed: !`git diff main...HEAD --name-only 2>/dev/null || git diff HEAD~5...HEAD --name-only`
- File count: !`git diff main...HEAD --name-only 2>/dev/null | wc -l || git diff HEAD~5...HEAD --name-only | wc -l`

## Your Task

Execute an iterative fix loop following this process:

### Step 1: Parse Configuration

Check if CLI arguments were provided. Arguments format:
```
--validators ddd-oop,dry,clean-code
--severity critical,high,medium
--iterations 5
```

**If NO arguments provided**, use AskUserQuestion to show interactive prompts:

**Prompt 1 - Validators (multi-select):**
- Header: "Validators"
- Question: "Which validators should run in the fix loop?"
- Options:
  - "DDD/OOP" - Anemic models, Tell/Don't Ask, method placement
  - "DRY" - Duplicated constants, logic, magic values
  - "Clean Code" - Function size, naming, SOLID principles
- Default: DDD/OOP and DRY selected
- multiSelect: true

**Prompt 2 - Severity (multi-select):**
- Header: "Severity"
- Question: "Which severity levels should trigger fixes?"
- Options:
  - "CRITICAL" - Must fix, blocks merge
  - "HIGH" - Should fix, high risk
  - "MEDIUM" - Address soon
  - "LOW" - Nice to have
- Default: CRITICAL and HIGH selected
- multiSelect: true

**Prompt 3 - Max Iterations:**
- Header: "Iterations"
- Question: "Maximum fix iterations before stopping?"
- Options:
  - "3" - Quick pass
  - "5 (Recommended)" - Balanced
  - "10" - Thorough
- Default: 5
- multiSelect: false

### Step 2: Initialize Loop State

Create a todo list to track the loop:

```
- [ ] Iteration 1: Validate and fix
- [ ] Iteration 2: Validate and fix (if needed)
- [ ] ... up to max iterations
- [ ] Generate final summary
```

Initialize variables:
- `iteration = 1`
- `totalFixed = 0`
- `previousIssueCount = Infinity`
- `stallCount = 0`

### Step 3: Run Iteration Loop

For each iteration until termination:

#### 3a. Display Iteration Header

```
═══════════════════════════════════════════════════════════════
 🔄 Iteration {n}/{max}
═══════════════════════════════════════════════════════════════

📋 Running validators: {selected validators}
   Scope: {file count} files changed vs main
```

#### 3b. Run Validators in Parallel

Use Task tool to launch selected validator agents **in parallel**:

**DDD/OOP Validator prompt:**
```
Validate the branch diff for DDD/OOP compliance.

Files to check:
{list of changed files}

Use: git diff main...HEAD (or HEAD~5 if no main)

Check for:
- Anemic domain models (data without behavior)
- Tell/Don't Ask violations (extracting data to decide externally)
- Method placement (logic in wrong object/service)
- Parameter bloat (too many primitives)

Output findings with:
- Severity: 🔴 CRITICAL / 🟠 HIGH / 🟡 MEDIUM / 🟢 LOW
- Location: 📍 file:line
- Recommendation: 💡 how to fix
```

**DRY Validator prompt:**
```
Validate the branch diff for DRY violations.

Files to check:
{list of changed files}

Use: git diff main...HEAD (or HEAD~5 if no main)

Check for:
- Duplicated constants/enums
- Repeated logic patterns
- Magic strings/numbers
- Similar code that should be unified

Output findings with:
- Severity: 🔴 CRITICAL / 🟠 HIGH / 🟡 MEDIUM / 🟢 LOW
- Location: 📍 file:line (for each occurrence)
- Recommendation: 💡 how to fix
```

**Clean Code Validator prompt:**
```
Validate the branch diff for clean code principles.

Files to check:
{list of changed files}

Use: git diff main...HEAD (or HEAD~5 if no main)

Check for:
- Long functions (>20 lines)
- Poor naming (unclear intent)
- SOLID violations
- Code smells (feature envy, god class, etc.)

Output findings with:
- Severity: 🔴 CRITICAL / 🟠 HIGH / 🟡 MEDIUM / 🟢 LOW
- Location: 📍 file:line
- Recommendation: 💡 how to fix
```

#### 3c. Filter and Count Findings

From validator outputs, extract findings matching selected severity levels.

Parse findings by looking for severity emoji patterns:
- 🔴 = CRITICAL
- 🟠 = HIGH
- 🟡 = MEDIUM
- 🟢 = LOW

Count: `currentIssueCount = number of matching findings`

#### 3d. Check Termination Conditions

**Condition 1: Success**
If `currentIssueCount == 0`:
```
───────────────────────────────────────────────────────────────
 ✅ No {severity levels} issues found!
───────────────────────────────────────────────────────────────
```
→ Go to Step 4 (Summary)

**Condition 2: Max Iterations**
If `iteration >= maxIterations`:
```
───────────────────────────────────────────────────────────────
 ⏹️ Max iterations reached. {currentIssueCount} issues remaining.
───────────────────────────────────────────────────────────────
```
→ Go to Step 4 (Summary)

**Condition 3: Stall Detection**
If `currentIssueCount >= previousIssueCount`:
  `stallCount++`
  If `stallCount >= 2`:
```
───────────────────────────────────────────────────────────────
 ⚠️ No progress for 2 iterations. Stopping to prevent infinite loop.
    {currentIssueCount} issues may require manual intervention.
───────────────────────────────────────────────────────────────
```
→ Go to Step 4 (Summary)

Otherwise, reset `stallCount = 0`.

#### 3e. Display Findings

```
───────────────────────────────────────────────────────────────
 Found {count} issues ({breakdown by severity})
───────────────────────────────────────────────────────────────

{List each finding with emoji, description, location, recommendation}
```

#### 3f. Apply Fixes

```
───────────────────────────────────────────────────────────────
 🔧 Fixing {count} issues...
───────────────────────────────────────────────────────────────
```

Use Task tool to launch fix-loop-fixer agent:

```
Apply fixes for the following validation findings:

{paste all findings with severity, location, recommendation}

Read each file, apply the recommended fix, and report what was changed.

Output format for each fix:
✅ Fixed: [description]
   📍 file:line
   📝 [explanation]

Or if skipped:
⚠️ Skipped: [reason]
```

Track: `totalFixed += number of fixes applied`

#### 3g. Continue Loop

```
→ Continuing to iteration {n+1}...
```

Update:
- `previousIssueCount = currentIssueCount`
- `iteration++`
- Mark current todo as completed, next as in_progress

Go back to Step 3a.

### Step 4: Generate Final Summary

```
═══════════════════════════════════════════════════════════════
 {✅ Fix Loop Complete | ⚠️ Fix Loop Stopped}
═══════════════════════════════════════════════════════════════

Iterations: {used}/{max}
Issues fixed: {totalFixed}
Remaining: {currentIssueCount}

{If remaining == 0}
🎉 No {severity levels} issues remaining!

{If remaining > 0}
📋 Remaining issues require manual review:
{list remaining issues}
```

## Validator Agent Mapping

| Selection | Agent (subagent_type) |
|-----------|----------------------|
| DDD/OOP | ddd-oop-validator |
| DRY | dry-violations-detector |
| Clean Code | clean-code-validator |

## Important Notes

- Always run validators in **parallel** using multiple Task tool calls
- The fixer agent is: `fix-loop-fixer`
- Scope is always **branch diff** (files changed vs main or HEAD~5)
- Stop if no progress for 2 consecutive iterations (prevents infinite loops)
- Be transparent: show what's being fixed before and after
