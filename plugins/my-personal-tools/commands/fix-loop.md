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

### Step 1: Discover Available Validators

Use Task tool to launch fix-loop-validator-discovery agent to find all available validators:

```
Pass to the agent:
- BUILTIN_MANIFEST: plugins/my-personal-tools/validators.json
- PLUGINS_DIR: <current project's installed plugins directory>
- PROJECT_DIR: <current project root>
```

The discovery agent will:
- Load built-in validators from validators.json
- Scan external plugins for validators in plugin.json
- Use fallback naming convention for *-validator agents
- Return a JSON list of all discovered validators with metadata

Capture the JSON output from the discovery agent and store it.

### Step 2: Parse CLI Arguments & Run Configurator

Check if CLI arguments were provided. Arguments format:
```
--validators ddd-oop,dry,react-best-practices
--severity critical,high
--iterations 5
```

**If CLI arguments provided:**
- Parse them (split by commas, convert to proper format)
- Skip to Step 2b (validation section)

**If NO CLI arguments provided:**
- Use Task tool to launch fix-loop-configurator agent
- Pass the discovered validators list from Step 1
- The configurator will invoke AskUserQuestion for three interactive prompts:
  1. Validator selection (multi-select)
  2. Severity filter (multi-select)
  3. Iteration count (single-select)
- Capture the configuration JSON output

### Step 2b: Validate & Merge Configuration

- Extract selectedValidators array from configurator output
- Validate that all selected validators were discovered in Step 1
- Warn if a validator is selected but not found: "⚠️ Validator X not found during discovery, skipping"
- Build final selectedValidators list with agent names from discovered metadata
- Store: severity levels, maxIterations
- Confirm configuration is ready to proceed

### Step 3: Initialize Loop State

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

### Step 4: Run Iteration Loop

For each iteration until termination:

#### 4a. Display Iteration Header

```
═══════════════════════════════════════════════════════════════
 🔄 Iteration {n}/{max}
═══════════════════════════════════════════════════════════════

📋 Running validators: {selected validators}
   Scope: {file count} files changed vs main
```

#### 4a.1: Log Selected Validators

Display which validators will run:

```
📋 Running validation loop with:
   Validators: {count} selected
     {validator name 1}
     {validator name 2}
   Severity: {CRITICAL, HIGH, etc.}
   Max iterations: {number}
```

#### 4b. Run Validators in Parallel

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

#### 4c. Filter and Count Findings

From all validator outputs, extract findings matching selected severity levels.

Parse findings using standardized regex patterns that work for ANY validator:

**Severity emoji + description regex:**
```
/(🔴|🟠|🟡|🟢)\s+([A-Z]+):\s+(.+)/
```
Extracts: emoji, severity level (CRITICAL/HIGH/MEDIUM/LOW), description

**Location regex:**
```
/📍\s+(.+):(\d+)/
```
Extracts: file path and line number

**Recommendation regex:**
```
/💡\s+(.+)/
```
Extracts: recommendation text

**Processing steps:**

For each validator output line:
1. Try to match against severity regex
2. If matches: extract emoji and map to severity level
   - 🔴 → CRITICAL
   - 🟠 → HIGH
   - 🟡 → MEDIUM
   - 🟢 → LOW
3. Check if severity level is in selectedSeverity
4. If yes: scan next lines for location and recommendation
   - Look for 📍 pattern for location
   - Look for 💡 pattern for recommendation
5. Combine into finding object: {severity, location, description, recommendation}
6. Add to findings list

**Error handling:**

If parsing fails for any line:
```
⚠️ WARNING: Could not parse finding from {validator_name}:
   {line_that_failed}
   (Skipping this finding but continuing with others)
```

Continue without stopping - prioritize fix-loop robustness.

**Counting:**

Count all successfully parsed findings that match selectedSeverity:
```
currentIssueCount = number of matching findings
```

Log the count with breakdown:
```
Found {currentIssueCount} issues:
  🔴 CRITICAL: {count}
  🟠 HIGH: {count}
  🟡 MEDIUM: {count}
  🟢 LOW: {count}
```

#### 4d. Check Termination Conditions

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

#### 4e. Display Findings

```
───────────────────────────────────────────────────────────────
 Found {count} issues ({breakdown by severity})
───────────────────────────────────────────────────────────────

{List each finding with emoji, description, location, recommendation}
```

#### 4f. Apply Fixes

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

#### 4g. Continue Loop

```
→ Continuing to iteration {n+1}...
```

Update:
- `previousIssueCount = currentIssueCount`
- `iteration++`
- Mark current todo as completed, next as in_progress

Go back to Step 3a.

### Step 5: Generate Final Summary

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
