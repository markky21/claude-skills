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

### Step 2c: Extract Test Configuration

From the configurator output (or defaults if skipped), extract:
- `generateTests` - boolean, true if user enabled test generation
- `testScope` - "all-changes" | "business-logic" | "flagged-code"
- `testCoverage` - "happy-paths-edges" | "happy-paths-only" | "comprehensive"

If user disabled test generation, set `generateTests = false` and skip all subsequent test steps.

If test generation enabled, proceed with test steps in the loop.

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

**CRITICAL SCOPE CONSTRAINT:**
- ONLY report findings on lines that appear in the git diff (lines starting with `+`)
- If an issue exists in a file but the code was NOT modified in this branch, do NOT report it
- The goal is to validate the CHANGES, not the entire codebase
- Before reporting any finding, verify the problematic code appears in the diff

Changed files:
{list of changed files}

Git diff to analyze:
{paste actual git diff main...HEAD output here}

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

**CRITICAL SCOPE CONSTRAINT:**
- ONLY report findings on lines that appear in the git diff (lines starting with `+`)
- If an issue exists in a file but the code was NOT modified in this branch, do NOT report it
- The goal is to validate the CHANGES, not the entire codebase
- Before reporting any finding, verify the problematic code appears in the diff

Changed files:
{list of changed files}

Git diff to analyze:
{paste actual git diff main...HEAD output here}

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

**CRITICAL SCOPE CONSTRAINT:**
- ONLY report findings on lines that appear in the git diff (lines starting with `+`)
- If an issue exists in a file but the code was NOT modified in this branch, do NOT report it
- The goal is to validate the CHANGES, not the entire codebase
- Before reporting any finding, verify the problematic code appears in the diff

Changed files:
{list of changed files}

Git diff to analyze:
{paste actual git diff main...HEAD output here}

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

#### 4b.1: Validate & Parse Validator Output

For each validator that completed:

1. **Check output format compliance**
   - Use fix-loop-output-validator agent to validate the actual output
   - Pass the validator's complete output text
   - Receive JSON validation result

2. **Log validation result**
   - If valid: log "✅ {validator_name}: output format valid"
   - If invalid: log "⚠️ {validator_name}: output format non-compliant, attempting to parse anyway"

3. **Proceed to parsing**
   - Both valid and invalid outputs proceed to Step 4c for parsing
   - Invalid outputs may have some findings skipped if unparseable
   - Non-blocking: one validator's format issues don't stop others

Example:
```
Validating outputs:
  ✅ ddd-oop-validator: output format valid
  ✅ dry-violations-detector: output format valid
  ⚠️ react-best-practices-validator: output format non-compliant
     (Parsing anyway - some findings may be skipped)
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
4. If yes: scan next lines to build complete finding
   - Scan forward from current line until:
     - You find a line with 📍 pattern (location)
     - You find a line with 💡 pattern (recommendation)
     - OR until the next severity emoji (🔴/🟠/🟡/🟢) appears
     - OR until end of validator output
   - If BOTH 📍 and 💡 patterns are found: create complete finding object
   - If EITHER pattern is missing: log warning and SKIP this finding
     ```
     ⚠️ INCOMPLETE: Finding missing required field in {validator_name}:
        {brief description}
        (Skipping - location and recommendation required)
     ```
5. Combine into finding only if COMPLETE: {severity, location, description, recommendation}
   - All four fields required
   - Add to findings list ONLY if complete
   - Incomplete findings logged and skipped

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
 From validators: {comma-separated list of validators that found issues}
───────────────────────────────────────────────────────────────

{For each finding, display in this format:}

{severity_emoji} {SEVERITY}: {description}
   📍 {validator_name}: {file}:{line}
   💡 {recommendation}

{blank line between findings}

{Example output:}

───────────────────────────────────────────────────────────────
 Found 8 issues (1 CRITICAL, 3 HIGH, 3 MEDIUM, 1 LOW)
 From validators: ddd-oop-validator, react-nextjs-validator
───────────────────────────────────────────────────────────────

🔴 CRITICAL: Hook called conditionally inside if statement
   📍 react-nextjs-validator: src/components/Form.tsx:42
   💡 Move useState call outside the conditional block. Hooks must be called unconditionally on every render.

🟠 HIGH: Multiple useState calls managing related state
   📍 react-nextjs-validator: src/hooks/useFormState.ts:15
   💡 Consider using useReducer instead of 5 related useState calls for better state management.

🟠 HIGH: Anemic domain model
   📍 ddd-oop-validator: src/domain/User.ts:8
   💡 Add behavior methods to the User entity instead of keeping it data-only.

🟡 MEDIUM: Derived state stored in component state
   📍 react-nextjs-validator: src/components/UserProfile.tsx:28
   💡 Calculate filteredUsers directly instead of storing in state. Use useMemo if computation is expensive.

[... more findings ...]
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

#### 4g. Generate Tests (if enabled)

If `generateTests == false`, skip to 4h.

Otherwise:

```
───────────────────────────────────────────────────────────────
 🧪 Generating tests for fixes...
───────────────────────────────────────────────────────────────
```

Use Task tool to launch fix-loop-test-generator agent:

**Inputs:**
- Git diff of changes (use: `git diff HEAD~{iteration}...HEAD`)
- Validator context (from previous validator runs, what each fix was about)
- Project patterns (detected from testing-patterns.json analysis)
- Test scope and coverage (from configuration)

**Output expected:**
- List of test files generated
- Count of test cases per file
- Skipped code sections with reasons

Example:

```
✅ Generated: src/domain/User.test.ts
   📝 Tests User entity creation, email validation, domain behavior
   📊 6 test cases covering constructor, validation, error paths

✅ Generated: src/services/UserRegistrationService.test.ts
   📝 Tests registration service with repository integration
   📊 8 test cases covering happy path, errors, state verification
```

#### 4h. Run Generated Tests (if enabled)

If `generateTests == false`, skip to 4i (continue loop).

Otherwise:

```
───────────────────────────────────────────────────────────────
 🏃 Running tests...
───────────────────────────────────────────────────────────────
```

Use Task tool to launch fix-loop-test-runner agent.

**Inputs:**
- Test framework type (auto-detect or from testing-patterns.json)
- Test file patterns to run

**Output expected:**
- Test results: pass/fail count
- Failed test names and error details
- Stack traces if available

Example output:

```
═══════════════════════════════════════════════════════════════
 🧪 Test Results
═══════════════════════════════════════════════════════════════

Total: 14 | ✅ Passed: 12 | ❌ Failed: 2

❌ User should throw error for invalid email
   AssertionError: expected no error to be thrown
   📍 src/domain/User.test.ts:52

❌ UserRegistrationService should reject duplicate users
   Error: repository.save not called
   📍 src/services/UserRegistrationService.test.ts:28
```

#### 4i. Check Test Status (if enabled)

If `generateTests == false`, go to 4j.

If all tests passed:
```
✅ All tests passed! Fixes are validated.
```
→ Proceed to Step 5 (Summary) - fix-loop is done

If some tests failed:
```
❌ {n} test failures detected. Analyzing...
```

Use Task tool to launch fix-loop-failure-analyzer agent.

**Inputs:**
- Test failure output (from test runner)
- Implementation code (files that failed tests)
- Validator context (why original changes were made)
- Generated test code

**Output expected:**
- For each failure: what test expected, what code did, what needs to be fixed
- Organized by failure category
- Implementation file references

Example:

```
🔴 TEST FAILURE: User should throw error for invalid email
   📍 src/domain/User.test.ts:52
   ❌ Expected: Constructor to throw "Invalid email"
   📊 Got: Constructor succeeded, created user
   💡 Fix needed: Call validateEmail() in constructor
   🔍 Context: DDD validator introduced validation logic
   📄 Implementation: src/domain/User.ts:5-12
```

#### 4j. Continue Loop or Terminate

Check termination conditions:

**Condition 1: All tests pass**
If all tests passed in 4i → Success, go to Step 5 (Summary)

**Condition 2: Test failures with iterations remaining**
If tests failed AND iteration < maxIterations:
```
→ Re-running validators with test failure context...
```
Pass to validators:
- Test failure analysis from 4i
- Implementation code that failed tests
- Original validator context

Validators will receive NEW context: tests expect X, implementation does Y, fix needed.

Reset: `iteration++`, go back to 4b (Run Validators) but with failure context

**Condition 3: Max iterations with failures**
If iteration >= maxIterations AND tests still failing:
```
⏹️ Max iterations reached with {n} failing tests.
```
→ Go to Step 5 (Summary) with warning

**Condition 4: Stall (no progress for 2 iterations)**
If same tests fail identically for 2 iterations:
```
⚠️ No test progress for 2 iterations. Stopping.
```
→ Go to Step 5 (Summary)

#### 4k. Continue Next Iteration

If continuing to next iteration:
- Update: `previousIssueCount` (for stall detection)
- Update: `iteration++`
- Mark todo completed, next marked as in_progress
- Go back to 4a (Iteration header)

#### 4l. Continue Loop

```
→ Continuing to iteration {n+1}...
```

Update:
- `previousIssueCount = currentIssueCount`
- `iteration++`
- Mark current todo as completed, next as in_progress

Go back to Step 4a.

### Step 5: Generate Final Summary

```
═══════════════════════════════════════════════════════════════
 {✅ Fix Loop Complete | ⚠️ Fix Loop Stopped}
═══════════════════════════════════════════════════════════════

Iterations: {used}/{max}
Issues fixed: {totalFixed}
Remaining issues: {currentIssueCount}

{If test generation was enabled:}
Tests Generated: {count}
Tests Status: {✅ All passing | ❌ {n} failing}

{If remaining issues > 0}
📋 Remaining issues require manual review:
{list remaining issues}

{If test generation enabled and tests failing}
⚠️ {n} tests still failing:
{list failing tests}
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
- Test generation is **optional** - can be disabled for faster iteration
- Test-driven fixing uses the same validators, just with new context (test failures)
- If tests fail consistently, likely indicates deeper issue requiring manual review
- Test generation agent will skip trivial code (getters, simple pass-throughs)
- Test output parsing tolerates some variation but warns on unparseable tests
