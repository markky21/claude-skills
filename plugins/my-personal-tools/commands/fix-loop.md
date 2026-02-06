---
description: Iteratively validate and fix code issues until no critical/high findings remain
allowed-tools: Bash(git *), Task, AskUserQuestion, TodoWrite, Read, Edit, Glob, Grep
---

# Fix Loop Command

Iteratively validates code and fixes issues until no critical/high severity findings remain (inspired by the [Ralph Wiggum technique](https://awesomeclaude.ai/ralph-wiggum)).

## Context Budget Rules

**The main orchestrator MUST NOT do any of the following:**

1. **Do NOT read changed files** with the Read tool — validators read files themselves in their own context windows
2. **Do NOT run `git diff` without `--name-only` or `--stat`** — full diff output consumes thousands of tokens in the main context
3. **Do NOT embed diff content in validator prompts** — Task tool prompts appear in the main context, so diff x N validators = context exhaustion
4. **Do NOT launch `fix-loop-output-validator` subagents** — use inline format checks instead

**WHY:** Task tool prompts are part of the main conversation context. A 500-line diff embedded in 9 validator prompts = 4,500 lines (~45K tokens) of duplicated text, which exhausts the context window before findings can even be processed.

**INSTEAD:** Tell each validator to run the diff command themselves (e.g., `Run: git diff main...HEAD`). Each validator has its own context window and can fetch whatever it needs independently.

## Step 0: Display Version Banner

**ALWAYS display this banner first, before anything else:**

```
═══════════════════════════════════════════════════════════════
 🔄 Fix Loop v1.15.0 (my-personal-tools plugin)
═══════════════════════════════════════════════════════════════
```

## Context

- Current branch: !`git branch --show-current`
- Base branch: !`git merge-base HEAD main 2>/dev/null && echo "main" || echo "HEAD~5"`
- Files changed: !`git diff main...HEAD --name-only 2>/dev/null || git diff HEAD~5...HEAD --name-only`
- File count: !`git diff main...HEAD --name-only 2>/dev/null | wc -l || git diff HEAD~5...HEAD --name-only | wc -l`
- Diff summary: !`git diff main...HEAD --stat 2>/dev/null || git diff HEAD~5...HEAD --stat`

> **WARNING:** Do NOT run `git diff` without `--name-only` or `--stat` in the main context. Full diff output belongs ONLY inside validator subagent context windows.

## Your Task

Execute an iterative fix loop following this process:

### Step 1: Check CLI Arguments or Run Interactive Configuration

Check if CLI arguments were provided. Arguments format:
```
--validators ddd,dry,clean,react,web
--severity critical,high
--iterations 5
--confirm 2
--tests disabled
--scope branch
```

**CLI Arguments Reference:**
| Argument | Values | Default | Description |
|----------|--------|---------|-------------|
| `--validators` | Comma-separated short codes | (interactive) | Which validators to run |
| `--severity` | critical,high,medium,low | critical,high | Severity levels to fix |
| `--iterations` | 1-10 | 5 | Max fix iterations |
| `--confirm` | 1-3 | 2 | Consecutive clean passes required |
| `--tests` | enabled/disabled | disabled | Generate tests for fixes |
| `--scope` | branch/uncommitted | branch | `branch`: `git diff main...HEAD`, `uncommitted`: `git diff HEAD` |

Use short codes from the Validator Agent Mapping table below.

**If CLI arguments provided:**
- Parse them (split by commas, convert to proper format)
- Skip to Step 2b (validation section)

**If NO CLI arguments provided:**
- Run interactive configuration directly using AskUserQuestion (NOT via subagent)
- Follow the prompts in Step 1a through Step 1e below

### Step 1a: Validator Group Selection (Architecture)

Use AskUserQuestion directly with these parameters:

**Question:** "Which architecture validators should run?"
**Header:** "Architecture"
**multiSelect:** true
**Options:**
- label: "DDD/OOP (ddd) (Recommended)"
  description: "Domain-Driven Design, anemic models, Tell/Don't Ask"
- label: "DRY Violations (dry) (Recommended)"
  description: "Duplicated code, magic values, repeated patterns"
- label: "Clean Code (clean)"
  description: "Function size, naming, SOLID principles"
- label: "API Design (api)"
  description: "REST/GraphQL patterns, resource naming, pagination"

### Step 1b: Validator Group Selection (Frontend)

Use AskUserQuestion directly:

**Question:** "Which frontend validators should run?"
**Header:** "Frontend"
**multiSelect:** true
**Options:**
- label: "React/Next.js (react)"
  description: "React 19+ and Next.js 15+ patterns, hooks"
- label: "Next.js Best Practices (next)"
  description: "RSC boundaries, async APIs, file conventions"
- label: "Vercel Performance (vercel)"
  description: "Async patterns, bundle optimization, hydration"
- label: "Vue.js (vue)"
  description: "Composition API, reactivity, computed, watchers"

### Step 1c: Validator Group Selection (UI/Styling)

Use AskUserQuestion directly:

**Question:** "Which UI/styling validators should run?"
**Header:** "UI/Styling"
**multiSelect:** true
**Options:**
- label: "Web Design Guidelines (web)"
  description: "Accessibility, UX patterns, WCAG compliance"
- label: "UI/UX Pro Max (uiux)"
  description: "Icons, interactions, contrast, layout"
- label: "Tailwind Design System (tailwind)"
  description: "Tailwind v4, @theme, design tokens, CVA"
- label: "Database Schema (db)"
  description: "Normalization, constraints, indexes, migrations"

### Step 1d: Severity and Iterations

Use AskUserQuestion directly:

**Question:** "Which severity levels should trigger fixes?"
**Header:** "Severity"
**multiSelect:** true
**Options:**
- label: "CRITICAL + HIGH (Recommended)"
  description: "Fix blocking and significant issues only"
- label: "All levels (CRITICAL, HIGH, MEDIUM, LOW)"
  description: "Fix everything including minor suggestions"
- label: "CRITICAL only"
  description: "Only fix fatal blocking issues"
- label: "CRITICAL, HIGH, MEDIUM"
  description: "Skip LOW suggestions"

### Step 1e: Iterations and Testing

Use AskUserQuestion directly:

**Question:** "Max iterations and test generation?"
**Header:** "Options"
**multiSelect:** false
**Options:**
- label: "5 iterations, no tests (Recommended)"
  description: "Balanced validation without test generation"
- label: "3 iterations, no tests"
  description: "Quick pass with minimal iterations"
- label: "5 iterations, with tests"
  description: "Generate tests for fixes (slower)"
- label: "10 iterations, with tests"
  description: "Thorough validation with test generation"

### Step 1f: Build Configuration

After collecting all selections, build the configuration:

1. Combine all selected validators from Steps 1a-1c
2. Map severity selection to array (e.g., "CRITICAL + HIGH" → ["CRITICAL", "HIGH"])
3. Extract iterations and test settings from Step 1e
4. Display configuration summary:

```
───────────────────────────────────────────────────────────────
 📋 Configuration Summary
───────────────────────────────────────────────────────────────

Validators: {count} selected
  {list each validator short code}

Severity: {levels}
Iterations: {max}
Tests: {enabled/disabled}

💡 Skip prompts next time with:
   /fix-loop --validators ddd,dry,clean --severity critical,high --iterations 5 --confirm 2 --tests disabled
───────────────────────────────────────────────────────────────
```

### Step 2b: Validate & Merge Configuration

- Extract selectedValidators array from configurator output
- Validate that all selected validators were discovered in Step 1
- Warn if a validator is selected but not found: "⚠️ Validator X not found during discovery, skipping"
- Build final selectedValidators list with agent names from discovered metadata
- Store: severity levels, maxIterations
- Derive `diffCmd` from `--scope` flag:
  - `branch` (default): `diffCmd = "git diff main...HEAD"`, `diffNameOnly = "git diff main...HEAD --name-only"`, `diffStat = "git diff main...HEAD --stat"`
  - `uncommitted`: `diffCmd = "git diff HEAD"`, `diffNameOnly = "git diff HEAD --name-only"`, `diffStat = "git diff HEAD --stat"`
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
- `confirmationLoops` - number of clean passes required (default: 2, from config)

Initialize **per-validator state** for each selected validator:

```
validatorStates = {}
for each selectedValidator:
  validatorStates[validator.name] = {
    status: "active",
    cleanPasses: 0,
    stallCount: 0,
    previousIssueCount: 0,
    retiredAtIteration: null
  }
```

Each validator tracks its own lifecycle independently. Validators retire individually when they reach their clean pass threshold or stall out. The loop ends when all validators are retired (or global max iterations hit as safety net).

### Step 4: Run Iteration Loop

For each iteration until termination:

#### 4a. Display Iteration Header

Determine which validators are still active:
```
activeValidators = all validators where validatorStates[name].status == "active"
retiredValidators = all validators where validatorStates[name].status != "active"
```

If `activeValidators` is empty → go to Step 5 (Summary).

Display iteration header:

```
═══════════════════════════════════════════════════════════════
 🔄 Iteration {n}/{max}
═══════════════════════════════════════════════════════════════

📋 Active validators: {active validator names}
   Retired: {count retired} ({list retired names with ✅/⚠️ status})
   Scope: {file count} files changed vs main
```

Example:
```
📋 Active validators: ddd-oop-validator, clean-code-validator
   Retired: 2 (✅ react-nextjs-validator, ⚠️ dry-violations-detector)
   Scope: 12 files changed vs main
```

#### 4b. Run Validators (with Batching)

Use Task tool to launch only **active** validator agents (skip retired validators entirely).

**Batching rule:** If 5+ validators are active, launch in batches of max 4 in parallel. Wait for each batch to complete before launching the next. If fewer than 5 are active, launch all in parallel.

**Validator Focus Areas:**

| Short Code | Focus | Check for |
|------------|-------|-----------|
| `ddd` | DDD/OOP compliance | Anemic domain models, Tell/Don't Ask violations, method placement, parameter bloat |
| `dry` | DRY violations | Duplicated constants/enums, repeated logic patterns, magic strings/numbers, code that should be unified |
| `clean` | Clean code principles | Long functions (>20 lines), poor naming, SOLID violations, code smells (feature envy, god class) |
| `react` | React/Next.js patterns | Hook rules, useState overuse, missing memoization, RSC boundary issues |
| `next` | Next.js best practices | File conventions, async APIs, metadata, error handling, route handlers |
| `vercel` | Vercel performance | Async patterns, bundle optimization, server/client patterns, hydration |
| `vue` | Vue.js patterns | Composition API, reactivity, computed, watchers, props, TypeScript |
| `web` | Web design guidelines | Accessibility, UX patterns, WCAG compliance, responsive design |
| `uiux` | UI/UX quality | Icons, interactions, contrast, layout, color consistency |
| `tailwind` | Tailwind design system | Tailwind v4, @theme, design tokens, CVA patterns, dark mode |
| `api` | API design principles | REST/GraphQL patterns, resource naming, pagination, error handling |
| `db` | Database schema design | Normalization, constraints, indexes, data types, migrations |

**Prompt template for ALL validators** (substitute `{focus}` and `{checks}` from table above):

```
Validate changed code for {focus}.
Scope: Run `{diffCmd}` to see the full diff of changes.
Changed files: {file_list}

Instructions:
1. Run `{diffCmd}` yourself to get the full diff
2. Use Read tool to examine changed files for full context
3. ONLY flag issues on changed lines (lines with `+` prefix in the diff)
4. Do NOT report issues in code that was not modified

Check for: {checks}

Output format (one block per finding):
{severity_emoji} {SEVERITY}: {description}
   📍 {file}:{line}
   💡 {recommendation}

Severity levels: 🔴 CRITICAL / 🟠 HIGH / 🟡 MEDIUM / 🟢 LOW
```

**Example** — launching DDD validator with `diffCmd = "git diff main...HEAD"`:

```
Validate changed code for DDD/OOP compliance.
Scope: Run `git diff main...HEAD` to see the full diff of changes.
Changed files: src/domain/User.ts, src/services/OrderService.ts, ...

Instructions:
1. Run `git diff main...HEAD` yourself to get the full diff
2. Use Read tool to examine changed files for full context
3. ONLY flag issues on changed lines (lines with `+` prefix in the diff)
4. Do NOT report issues in code that was not modified

Check for: Anemic domain models, Tell/Don't Ask violations, method placement, parameter bloat

Output format (one block per finding):
{severity_emoji} {SEVERITY}: {description}
   📍 {file}:{line}
   💡 {recommendation}

Severity levels: 🔴 CRITICAL / 🟠 HIGH / 🟡 MEDIUM / 🟢 LOW
```

#### 4b.1: Inline Output Format Check

For each validator that completed, do a quick inline check (NO subagent):

1. **Scan output** for severity emojis (🔴, 🟠, 🟡, 🟢) and location markers (📍)
2. **Log result:**
   - If emojis and locations found: `✅ {validator_name}: output parseable`
   - If no emojis found: `ℹ️ {validator_name}: no findings (clean)`
   - If emojis found but no locations: `⚠️ {validator_name}: findings lack locations, parsing may skip some`
3. **Proceed to Step 4c** — both valid and malformed outputs go to regex parsing

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

**Counting (per-validator):**

Group all successfully parsed findings **by source validator**, then count per-validator:
```
for each activeValidator:
  validatorFindings[name] = findings from this validator matching selectedSeverity
  validatorIssueCount[name] = count of those findings
```

Also compute total for display:
```
currentIssueCount = sum of all validatorIssueCount values
```

Log the total with breakdown:
```
Found {currentIssueCount} issues:
  🔴 CRITICAL: {count}
  🟠 HIGH: {count}
  🟡 MEDIUM: {count}
  🟢 LOW: {count}
```

#### 4d. Update Per-Validator State and Check Termination

For each **active** validator, count its findings from Step 4c and update its state:

```
for each activeValidator:
  issueCount = count of findings from this validator matching selectedSeverity

  if issueCount == 0:
    # Clean pass
    validatorStates[name].cleanPasses++
    validatorStates[name].stallCount = 0

    if validatorStates[name].cleanPasses >= confirmationLoops:
      validatorStates[name].status = "retired-clean"
      validatorStates[name].retiredAtIteration = iteration
      log: "✅ {name}: Retired clean ({cleanPasses}/{confirmationLoops} passes) at iteration {iteration}"
    else:
      log: "{name}: clean pass {cleanPasses}/{confirmationLoops}"

  else:
    # Found issues - reset clean counter
    validatorStates[name].cleanPasses = 0

    # Stall detection: same issue count as last run?
    if issueCount == validatorStates[name].previousIssueCount:
      validatorStates[name].stallCount++
      if validatorStates[name].stallCount >= 2:
        validatorStates[name].status = "retired-stalled"
        validatorStates[name].retiredAtIteration = iteration
        log: "⚠️ {name}: Retired stalled at iteration {iteration} ({issueCount} issues remaining)"
    else:
      validatorStates[name].stallCount = 0

    validatorStates[name].previousIssueCount = issueCount
```

Display per-validator status summary:

```
───────────────────────────────────────────────────────────────
 Validator Status (Iteration {n}):
   {for each validator, show one of:}
   ✅ {name}: Retired clean ({cleanPasses}/{confirmationLoops}) at iteration {n}
   ⚠️ {name}: Retired stalled at iteration {n} ({issueCount} remaining)
   🔄 {name}: {issueCount} issues (clean {cleanPasses}/{confirmationLoops}, stall {stallCount}/2)
   🔄 {name}: clean pass {cleanPasses}/{confirmationLoops}
───────────────────────────────────────────────────────────────
```

**Check termination conditions:**

**Condition 1: All Validators Retired**
If no validators have `status == "active"`:
```
───────────────────────────────────────────────────────────────
 ✅ All validators retired.
───────────────────────────────────────────────────────────────
```
→ Go to Step 5 (Summary)

**Condition 2: Max Iterations**
If `iteration >= maxIterations`:
```
───────────────────────────────────────────────────────────────
 ⏹️ Max iterations reached. {count active} validators still active.
───────────────────────────────────────────────────────────────
```
→ Go to Step 5 (Summary)

Otherwise → continue to Step 4e with findings from active validators that had issues.

#### 4e. Display Findings

Only display findings from **active** validators that had issues this iteration (skip retired validators):

```
───────────────────────────────────────────────────────────────
 Found {count} issues ({breakdown by severity})
 From active validators: {comma-separated list of active validators that found issues}
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

Only pass findings from **active** validators that had issues this iteration to the fixer (not retired validators):

```
───────────────────────────────────────────────────────────────
 🔧 Fixing {count} issues from {active validator count} active validators...
───────────────────────────────────────────────────────────────
```

Use Task tool to launch `my-personal-tools:fix-loop-fixer` agent.

**Fixer prompt must contain ONLY structured findings** (3 lines per finding: severity, location, recommendation). Do NOT include file contents, diff output, or validator raw output in the fixer prompt.

```
Apply fixes for the following validation findings.
Use Read tool to examine each file before fixing. Do NOT include file contents or diff output in your response.

Findings to fix:
{for each finding, exactly 3 lines:}
{severity_emoji} {SEVERITY}: {description}
   📍 {file}:{line}
   💡 {recommendation}

Output format for each fix:
✅ Fixed: [description]
   📍 file:line
   📝 [brief explanation of what changed]

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

Use Task tool to launch `my-personal-tools:fix-loop-test-generator` agent:

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

Use Task tool to launch `my-personal-tools:fix-loop-test-runner` agent.

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

Use Task tool to launch `my-personal-tools:fix-loop-failure-analyzer` agent.

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
If tests failed AND iteration < maxIterations AND active validators remain:
```
→ Re-running active validators with test failure context...
```
Pass to active validators only:
- Test failure analysis from 4i
- Implementation code that failed tests
- Original validator context

Validators will receive NEW context: tests expect X, implementation does Y, fix needed.

Reset: `iteration++`, go back to 4b (Run active Validators) but with failure context

**Condition 3: Max iterations with failures**
If iteration >= maxIterations AND tests still failing:
```
⏹️ Max iterations reached with {n} failing tests.
```
→ Go to Step 5 (Summary) with warning

**Condition 4: All validators retired**
If no active validators remain:
```
All validators retired. {n} tests still failing.
```
→ Go to Step 5 (Summary)

#### 4k. Continue Next Iteration

If continuing to next iteration:
- Update: `iteration++`
- Mark todo completed, next marked as in_progress
- Go back to 4a (Iteration header)

Note: per-validator `previousIssueCount` is already updated in Step 4d.

### Step 5: Generate Final Summary

Determine overall status:
- If all validators are `retired-clean` → `✅ Fix Loop Complete`
- If any validator is `retired-stalled` or still `active` (max iterations) → `⚠️ Fix Loop Stopped`

```
═══════════════════════════════════════════════════════════════
 {✅ Fix Loop Complete | ⚠️ Fix Loop Stopped}
═══════════════════════════════════════════════════════════════

Iterations: {used}/{max}
Issues fixed: {totalFixed}

Validator Results:
  {for each validator, show:}
  ✅ {name}: Retired clean ({cleanPasses}/{confirmationLoops} passes) at iteration {n}
  ⚠️ {name}: Retired stalled at iteration {n} ({previousIssueCount} issues remaining)
  🔄 {name}: Still active ({previousIssueCount} issues, max iterations reached)

{If any validator is retired-stalled or still active:}
📋 Remaining issues require manual review:
{list last findings from stalled/active validators}

{If test generation was enabled:}
Tests Generated: {count}
Tests Status: {✅ All passing | ❌ {n} failing}

{If test generation enabled and tests failing}
⚠️ {n} tests still failing:
{list failing tests}
```

## Validator Agent Mapping

| CLI Short Code | Agent (subagent_type) |
|----------------|----------------------|
| `ddd` | my-personal-tools:ddd-oop-validator |
| `dry` | my-personal-tools:dry-violations-detector |
| `clean` | my-personal-tools:clean-code-validator |
| `react` | my-personal-tools:react-nextjs-validator |
| `web` | my-personal-tools:web-design-guidelines-validator |
| `tailwind` | my-personal-tools:tailwind-design-system-validator |
| `uiux` | my-personal-tools:ui-ux-pro-max-validator |
| `vercel` | my-personal-tools:vercel-react-best-practices-validator |
| `api` | my-personal-tools:api-design-principles-validator |
| `db` | my-personal-tools:database-schema-designer-validator |
| `next` | my-personal-tools:next-best-practices-validator |
| `vue` | my-personal-tools:vue-best-practices-validator |

## Severity Mapping

| CLI Code | Internal |
|----------|----------|
| `critical` | CRITICAL |
| `high` | HIGH |
| `medium` | MEDIUM |
| `low` | LOW |

## Important Notes

- Always run **active** validators in **parallel** using multiple Task tool calls (skip retired validators)
- **Batching:** If 5+ validators are active, launch in batches of max 4 in parallel. Wait for each batch to complete before launching the next
- Each validator tracks its own lifecycle: `cleanPasses`, `stallCount`, `previousIssueCount`, `status`
- A validator retires clean after `confirmationLoops` consecutive clean passes (default: 2)
- A validator retires stalled after 2 consecutive iterations with the same issue count
- If a validator had clean passes but then finds new issues (e.g. introduced by another fix), its `cleanPasses` resets to 0
- The loop ends when all validators are retired OR global max iterations is reached
- The fixer agent is: `my-personal-tools:fix-loop-fixer`
- **Scope:** Controlled by `--scope` flag. `branch` (default) = `git diff main...HEAD`, `uncommitted` = `git diff HEAD`
- Be transparent: show per-validator status after each iteration
- Test generation is **optional** - can be disabled for faster iteration
- Test-driven fixing uses the same active validators, just with new context (test failures)
- If tests fail consistently, likely indicates deeper issue requiring manual review
- Test generation agent will skip trivial code (getters, simple pass-throughs)
- Test output parsing tolerates some variation but warns on unparseable tests

### Context Budget Reminders

These rules are **critical** for preventing context window exhaustion when running many validators:

1. **NEVER read changed files** in the main context — validators have their own context windows and read files themselves
2. **NEVER run `git diff` without `--name-only` or `--stat`** in the main context — full diff output wastes thousands of tokens
3. **NEVER embed diff content in validator prompts** — tell validators to run `{diffCmd}` themselves
4. **NEVER use `fix-loop-output-validator` subagents** — use inline emoji/location scan instead
5. **ALWAYS use the compact prompt template** from Step 4b — ~15 lines per validator, not ~(30 + diff_size)
6. **ALWAYS pass only structured findings** (3 lines each) to the fixer — no raw output, no file contents
