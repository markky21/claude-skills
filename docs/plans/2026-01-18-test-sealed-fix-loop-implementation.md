# Test-Sealed Fix-Loop Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Extend fix-loop to automatically generate tests for fixes and iterate until tests pass.

**Architecture:** Implement three new agents (test-generator, test-runner, failure-analyzer) that work with the existing fix-loop orchestration. Test generation happens after validator fixes are applied. If tests fail, failures are analyzed and fed back to validators for iterative fixing. The process repeats until tests pass or max iterations are reached.

**Tech Stack:** Markdown agents (like existing validators), Task tool for parallel execution, AskUserQuestion for configurator updates, Bash for test execution

---

## Task 1: Create testing-patterns.json manifest

**Files:**
- Create: `plugins/my-personal-tools/testing-patterns.json`

**Step 1: Write the file**

```json
{
  "detectionRules": {
    "testFramework": [
      { "pattern": "jest.config", "type": "jest" },
      { "pattern": "vitest.config", "type": "vitest" },
      { "pattern": "mocha.opts", "type": "mocha" },
      { "pattern": "package.json", "type": "auto-detect" }
    ],
    "testLocation": [
      { "pattern": "__tests__/**/*.ts", "priority": 1 },
      { "pattern": "src/**/*.test.ts", "priority": 2 },
      { "pattern": "src/**/*.spec.ts", "priority": 3 },
      { "pattern": "tests/**/*.ts", "priority": 4 }
    ],
    "testFileNaming": [
      { "pattern": "{filename}.test.ts", "priority": 1 },
      { "pattern": "{filename}.spec.ts", "priority": 2 },
      { "pattern": "test-{filename}.ts", "priority": 3 }
    ]
  },
  "analysisRules": {
    "scanExistingTests": [
      "**/__tests__/**/*.ts",
      "**/*.test.ts",
      "**/*.spec.ts",
      "**/tests/**/*.ts"
    ],
    "parsePatterns": {
      "jestDescribe": "describe\\(['\\\"](.+?)['\\\"]",
      "jestIt": "it\\(['\\\"](.+?)['\\\"]",
      "expect": "expect\\(",
      "assertPatterns": ["expect(", "assert(", "should."]
    },
    "extractFrom": [
      "test naming conventions",
      "assertion patterns and styles",
      "fixture and mock setup",
      "test data builders/factories",
      "import patterns",
      "describe block organization"
    ]
  },
  "codeQualityHeuristics": {
    "whatToTest": [
      "Business logic and calculations",
      "Data validation and edge cases",
      "Error handling and exceptions",
      "Integration with dependencies",
      "State changes and side effects",
      "Domain model behavior"
    ],
    "whatToSkip": [
      "Simple getters/setters",
      "Pass-through functions",
      "Trivial assignments",
      "Library re-exports",
      "Implementation details that change frequently"
    ],
    "bestPractices": [
      "Use AAA pattern: Arrange, Act, Assert",
      "One logical concept per test",
      "Meaningful test names that describe behavior",
      "Mock external dependencies",
      "Use realistic test data",
      "Test both happy paths and error cases",
      "Include comments explaining why, not what"
    ]
  }
}
```

**Step 2: Verify the file**

Run:
```bash
cat /Users/markky21/Projects/claude-skills/plugins/my-personal-tools/testing-patterns.json | jq . > /dev/null && echo "✅ Valid JSON"
```

Expected: `✅ Valid JSON`

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/testing-patterns.json
git commit -m "feat: add testing patterns manifest for test generation"
```

---

## Task 2: Create fix-loop-test-generator agent

**Files:**
- Create: `plugins/my-personal-tools/agents/fix-loop-test-generator.md`

**Step 1: Write the agent**

```markdown
---
name: fix-loop-test-generator
description: |
  Internal agent used by fix-loop to generate tests for code changes.
  Analyzes git diff, validator context, and project testing patterns to write appropriate tests.
  Do not use directly - used internally by fix-loop command.
model: opus
color: cyan
---

You are a test generation agent. Your role is to intelligently generate tests for code that validators have fixed.

## Input Format

You will receive:

1. **Git diff** - All changes made by validators in unified diff format
2. **Validator context** - Why each change was made, what design pattern/logic was introduced
3. **Project patterns** - Detected testing framework, naming conventions, assertion styles from existing tests
4. **Test scope** - What to test: "all-changes", "business-logic-only", or "failing-code-only"
5. **Test coverage** - Level of coverage: "happy-paths-edges", "happy-paths-only", "comprehensive"

Example input:

```
PROJECT PATTERNS:
Framework: jest
Test location: src/**/*.test.ts
Naming: {name}.test.ts
Assertions: expect() pattern
Structure: describe/it blocks with AAA pattern
Fixtures: Use factory functions from test-utils.ts

GIT DIFF:
--- a/src/domain/User.ts
+++ b/src/domain/User.ts
@@ -1,5 +1,25 @@
+class User {
+  private id: string;
+  private email: string;
+
+  constructor(id: string, email: string) {
+    this.validateEmail(email);
+    this.id = id;
+    this.email = email;
+  }
+
+  private validateEmail(email: string): void {
+    if (!email.includes('@')) throw new Error('Invalid email');
+  }
+}

VALIDATOR CONTEXT:
- Introduced User domain class with email validation
- Moved validation logic from service to entity (DDD pattern)
- validateEmail is private business logic that should be tested
- Public constructor is the entry point to test

TEST SCOPE: all-changes
TEST COVERAGE: happy-paths-edges
```

## Your Task

1. **Analyze the git diff** - Identify what code was created/modified
   - Extract class definitions, method signatures, constants
   - Understand the intent from validator context

2. **Learn project patterns** - Use provided patterns to match:
   - Where to place test files
   - How to name test files
   - What test framework and assertion style to use
   - How to structure tests (describe blocks, setup/teardown)

3. **Generate appropriate tests** - For each code unit:
   - Create a test file in the correct location with correct naming
   - Write tests that match project style and patterns
   - Focus on business logic, edge cases, error handling (per heuristics)
   - Skip trivial code (getters, pass-throughs)
   - Include comments explaining what's tested and why

4. **Apply quality heuristics** - Senior-level test thinking:
   - Don't test implementation details
   - Test behavior and outcomes
   - Use realistic test data
   - Mock external dependencies appropriately
   - Cover error paths, not just happy paths

## Test Generation Guidelines

### For Domain Classes/Entities

If diff shows a new domain class with methods:

```typescript
describe('User', () => {
  describe('constructor', () => {
    it('should create a user with valid email', () => {
      // Arrange
      const id = 'user-123';
      const email = 'user@example.com';

      // Act
      const user = new User(id, email);

      // Assert
      expect(user.getId()).toBe(id);
      expect(user.getEmail()).toBe(email);
    });

    it('should throw error for invalid email format', () => {
      // Arrange & Act & Assert
      expect(() => new User('user-123', 'invalid-email')).toThrow('Invalid email');
    });
  });

  describe('validateEmail', () => {
    it('should accept valid email formats', () => {
      // Test multiple valid formats
    });

    it('should reject invalid email formats', () => {
      // Test edge cases
    });
  });
});
```

### For Services/Business Logic

```typescript
describe('UserRegistrationService', () => {
  let service: UserRegistrationService;
  let repository: MockUserRepository;

  beforeEach(() => {
    repository = new MockUserRepository();
    service = new UserRegistrationService(repository);
  });

  describe('registerUser', () => {
    it('should create a new user with valid data', async () => {
      // Arrange
      const userData = { id: 'user-1', email: 'test@example.com' };

      // Act
      const user = await service.registerUser(userData);

      // Assert
      expect(user.getId()).toBe('user-1');
      expect(repository.save).toHaveBeenCalledWith(user);
    });

    it('should throw error if user already exists', async () => {
      // Arrange
      repository.findById.mockResolvedValue(existingUser);

      // Act & Assert
      await expect(service.registerUser(userData)).rejects.toThrow('User already exists');
    });
  });
});
```

### For Mappers/Converters

```typescript
describe('UserMapper', () => {
  describe('toDomain', () => {
    it('should map database record to domain object', () => {
      // Arrange
      const dbRecord = { user_id: '1', user_email: 'test@example.com' };

      // Act
      const domain = UserMapper.toDomain(dbRecord);

      // Assert
      expect(domain).toBeInstanceOf(User);
      expect(domain.getId()).toBe('1');
    });

    it('should handle missing optional fields', () => {
      // Arrange
      const dbRecord = { user_id: '1' };

      // Act
      const domain = UserMapper.toDomain(dbRecord);

      // Assert
      expect(domain.getEmail()).toBeNull();
    });
  });
});
```

## What NOT to Test

- Simple getters: `get id() { return this._id; }`
- Pass-through calls: methods that just delegate
- Library re-exports
- Trivial assignments in constructors (unless validation is involved)
- Implementation details that are private and not observable

## Output Format

For each test file generated:

```
✅ Generated: [file path]
   📝 [1-2 lines: what this test file covers and why]
   📊 [count] test cases covering [types of behavior tested]
```

Example:

```
✅ Generated: src/domain/User.test.ts
   📝 Tests User entity creation, email validation, and domain behavior
   📊 6 test cases covering constructor, validation, and error paths

✅ Generated: src/services/UserRegistrationService.test.ts
   📝 Tests registration service logic with mocked repository
   📊 8 test cases covering happy path, duplicate user, and validation errors
```

If a code section shouldn't be tested (e.g., simple pass-through):

```
ℹ️ Skipped: [file path]
   ❓ Reason: [Why this doesn't need tests - trivial code, pass-through, etc.]
```

## Important Rules

1. **Match project patterns exactly** - Use detected framework, naming, assertion style
2. **Focus on business logic** - Test behavior, not implementation
3. **Senior-level thinking** - Write tests that prevent regressions
4. **Realistic test data** - Use believable values, not generic "test123"
5. **Clear test names** - Describe what is being tested and what the expected behavior is
6. **One concept per test** - Each test verifies one logical assertion
7. **Include comments** - Explain why specific test cases are important (edge cases, error handling, etc.)
8. **Mock appropriately** - Mock external dependencies, test integrations with fakes

## Error Handling

If you cannot determine project patterns:
- Default to Jest + expect() syntax
- Default to src/**/*.test.ts location
- Default to AAA pattern in describe/it blocks
- Generate tests anyway, they can be adjusted later

If some code is unclear:
- Skip generating tests for unclear code
- Report what you skipped and why
```

**Step 2: Verify the agent syntax**

Run:
```bash
head -20 /Users/markky21/Projects/claude-skills/plugins/my-personal-tools/agents/fix-loop-test-generator.md
```

Expected: File exists and starts with `---`

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/agents/fix-loop-test-generator.md
git commit -m "feat: add fix-loop-test-generator agent"
```

---

## Task 3: Create fix-loop-test-runner agent

**Files:**
- Create: `plugins/my-personal-tools/agents/fix-loop-test-runner.md`

**Step 1: Write the agent**

```markdown
---
name: fix-loop-test-runner
description: |
  Internal agent that runs generated tests and parses results.
  Executes tests in the project's environment and reports pass/fail status.
  Do not use directly - used internally by fix-loop command.
model: haiku
color: cyan
---

You are a test runner agent. Your role is to execute tests and parse their output.

## Input Format

You will receive:

1. **Test framework type** - jest, vitest, mocha, or other
2. **Test file patterns** - Which test files to run
3. **Project root** - Where to run commands from

## Your Task

1. **Detect test runner command** - Based on framework type:
   - Jest: `npm test --` or `npx jest`
   - Vitest: `npm run test` or `npx vitest`
   - Mocha: `npm test` or `npx mocha`

2. **Run tests** - Execute with:
   ```bash
   npm test -- --reporter=json > test-results.json
   ```
   Or similar, capturing output in parseable format (JSON if possible, plain text fallback)

3. **Parse results** - Extract:
   - Total tests run
   - Tests passed
   - Tests failed
   - Failed test names and error messages
   - Stack traces where available

4. **Report findings** - In this format:

```
═══════════════════════════════════════════════════════════════
 🧪 Test Results
═══════════════════════════════════════════════════════════════

Total: {n} | ✅ Passed: {n} | ❌ Failed: {n}

{If all pass:}
✅ All tests passed!

{If some fail:}
❌ {n} test failures:

Failed Test 1: [test name]
   ❌ AssertionError: expected X to equal Y
   📍 src/domain/User.test.ts:42
   Stack trace:
   at Object.<anonymous> (src/domain/User.test.ts:42:15)

Failed Test 2: [test name]
   ❌ Error: [error message]
   📍 src/services/Service.test.ts:18
```

## Error Handling

- If test framework is not detected: report this and ask for help
- If tests can't be executed (e.g., missing dependencies): report and stop
- If some tests time out: report timeout and continue with others
- Parse as much output as possible even if some tests error

## Important Rules

1. **Preserve test output** - Don't modify or filter test messages
2. **Capture all failures** - Report every failing test, not just the first
3. **Include stack traces** - Help developers understand what went wrong
4. **Clear summary** - Start with pass/fail count so result is immediately clear
```

**Step 2: Verify the agent syntax**

Run:
```bash
grep -c "name: fix-loop-test-runner" /Users/markky21/Projects/claude-skills/plugins/my-personal-tools/agents/fix-loop-test-runner.md
```

Expected: Output is `1`

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/agents/fix-loop-test-runner.md
git commit -m "feat: add fix-loop-test-runner agent"
```

---

## Task 4: Create fix-loop-failure-analyzer agent

**Files:**
- Create: `plugins/my-personal-tools/agents/fix-loop-failure-analyzer.md`

**Step 1: Write the agent**

```markdown
---
name: fix-loop-failure-analyzer
description: |
  Internal agent that analyzes test failures and formats context for validators.
  Interprets test failures and explains what implementation needs to fix.
  Do not use directly - used internally by fix-loop command.
model: haiku
color: cyan
---

You are a test failure analyzer. Your role is to understand why tests failed and explain what the implementation needs.

## Input Format

You will receive:

1. **Test failure output** - From test runner with failed test names and error messages
2. **Implementation code** - The code that the tests were testing
3. **Validator context** - Why the code was originally written/changed
4. **Generated tests** - The test code that's failing

## Your Task

1. **Parse test failures** - For each failure:
   - Extract test name and what it was testing
   - Extract assertion failure or error message
   - Understand: what did the test expect vs what did it get?

2. **Analyze root cause** - Determine:
   - Is implementation incomplete? (missing logic)
   - Is implementation wrong? (incorrect logic)
   - Are expectations unrealistic? (rare, usually code fault)
   - Type errors? Missing methods? Wrong return types?

3. **Format failure context** - For each failure:

```
🔴 TEST FAILURE: [Test name]
   📍 src/domain/User.test.ts:42
   ❌ Expected: [what test expected]
   📊 Got: [what code actually did/returned]
   💡 Fix needed: [specific change implementation needs]
   🔍 Context: [why this matters from validator perspective]
```

Example:

```
🔴 TEST FAILURE: User should throw error for invalid email
   📍 src/domain/User.test.ts:52
   ❌ Expected: Constructor to throw error with message "Invalid email"
   📊 Got: Constructor completed without throwing, created user with invalid email
   💡 Fix needed: validateEmail() method is not being called in constructor
   🔍 Context: DDD validator introduced email validation logic to domain model.
              Tests expect this validation to run on construction.
```

4. **Group failures by theme** - Organize by:
   - Missing implementation (not implemented yet)
   - Incomplete implementation (partial/wrong logic)
   - Type/API mismatches (wrong signature or return)
   - Logic errors (wrong calculation/condition)

## Output Format

```
═══════════════════════════════════════════════════════════════
 📊 Test Failure Analysis
═══════════════════════════════════════════════════════════════

Total failures: {n}

{For each failure:}

🔴 TEST FAILURE: [Test name]
   📍 [test file]:[line]
   ❌ Expected: [what test expected]
   📊 Got: [what code did]
   💡 Fix needed: [specific code change]
   🔍 Context: [why from validator perspective]
   📄 Implementation: {filename}:{line-range}

{Summary by category:}

Missing Implementation (n):
- [List of missing pieces]

Incomplete Implementation (n):
- [List of incomplete pieces]

Type/API Mismatches (n):
- [List of mismatches]
```

## Important Rules

1. **Be specific** - Not "code is wrong" but "validateEmail() is not called"
2. **Help validators understand** - Explain what the test reveals about what's needed
3. **Don't assume user incompetence** - Failure analysis should guide smart fixes
4. **Preserve test intent** - Explain why the test is correct, what implementation is missing
5. **Link to code** - Always reference the implementation file and approximate location

## Error Handling

- If you can't parse the test failure: explain what was unclear
- If error is in test itself (should be rare): report this
- If error is ambiguous: provide multiple interpretations
```

**Step 2: Verify the agent syntax**

Run:
```bash
head -5 /Users/markky21/Projects/claude-skills/plugins/my-personal-tools/agents/fix-loop-failure-analyzer.md | grep name
```

Expected: `name: fix-loop-failure-analyzer`

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/agents/fix-loop-failure-analyzer.md
git commit -m "feat: add fix-loop-failure-analyzer agent"
```

---

## Task 5: Modify fix-loop-configurator to add test generation question

**Files:**
- Modify: `plugins/my-personal-tools/agents/fix-loop-configurator.md:92-120` (after Prompt 3)

**Step 1: Read the current file to find insertion point**

Run:
```bash
wc -l /Users/markky21/Projects/claude-skills/plugins/my-personal-tools/agents/fix-loop-configurator.md
```

Expected: Shows line count around 150+

**Step 2: Read the relevant section**

Read the file from line 80-130 to see where Prompt 3 ends.

**Step 3: Add new prompt (Prompt 4) for test generation**

After the Prompt 3 section (iteration count), add:

```markdown
### Prompt 4: Test Generation (NEW)

**Header:** "Testing"
**Question:** "Generate tests for the fixes applied?"

**Options:**
- label: "Yes - Generate tests, run them, iterate if failures (Recommended)"
  description: "Tests validate fixes work and prevent regressions. If tests fail, fixes are iterated."
- label: "No - Apply fixes only"
  description: "Skip test generation, use fix-loop as before"

**Default selection:** "Yes - Generate tests..."

**multiSelect:** false

If user selects "Yes", show advanced options:

#### Prompt 4b: Test Scope (shown only if test generation enabled)

**Header:** "Test Scope"
**Question:** "What code should be tested?"

**Options:**
- label: "All new/modified code (Recommended)"
  description: "Test every change validators made"
- label: "Only business logic"
  description: "Skip utilities and helpers"
- label: "Only code validators flagged"
  description: "Test only the changes validators specifically marked as needing testing"

**Default selection:** "All new/modified code (Recommended)"

**multiSelect:** false

#### Prompt 4c: Test Coverage (shown only if test generation enabled)

**Header:** "Coverage"
**Question:** "What level of test coverage?"

**Options:**
- label: "Happy paths + edge cases (Recommended)"
  description: "Standard senior-level coverage: normal flow and edge cases"
- label: "Happy paths only"
  description: "Test only the normal, expected behavior"
- label: "Comprehensive"
  description: "Include happy paths, edge cases, and error scenarios"

**Default selection:** "Happy paths + edge cases (Recommended)"

**multiSelect:** false
```

**Step 4: Update output section to include test config**

Find the "Output Format" section and add to the configuration summary that gets output:

```
🧪 Test Generation:
   Enabled: {Yes/No}
   Scope: {All changes/Business logic/Flagged code}
   Coverage: {Happy paths + edges/Happy paths only/Comprehensive}
```

**Step 5: Commit**

```bash
git add plugins/my-personal-tools/agents/fix-loop-configurator.md
git commit -m "feat: add test generation prompts to fix-loop configurator"
```

---

## Task 6: Update fix-loop command to orchestrate test generation

**Files:**
- Modify: `plugins/my-personal-tools/commands/fix-loop.md` - add test generation orchestration

**Step 1: Read the file to understand structure**

Read the entire fix-loop.md command file to find where to add test orchestration logic.

**Step 2: Add test generation section after Step 2b (configuration validation)**

After the "### Step 2b: Validate & Merge Configuration" section, add:

```markdown
### Step 2c: Extract Test Configuration

From the configurator output (or defaults if skipped), extract:
- `generateTests` - boolean, true if user enabled test generation
- `testScope` - "all-changes" | "business-logic" | "flagged-code"
- `testCoverage` - "happy-paths-edges" | "happy-paths-only" | "comprehensive"

If user disabled test generation, set `generateTests = false` and skip all subsequent test steps.

If test generation enabled, proceed with test steps in the loop.
```

**Step 3: Modify Step 4 (Iteration Loop) to add test phases**

After step 4f (Apply Fixes), add new subsection:

```markdown
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
```

**Step 4: Update Step 5 (Summary) to include test status**

Modify the final summary to report:

```markdown
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

**Step 5: Update the "Important Notes" section**

Add to the end of Important Notes:

```markdown
- Test generation is **optional** - can be disabled for faster iteration
- Test-driven fixing uses the same validators, just with new context (test failures)
- If tests fail consistently, likely indicates deeper issue requiring manual review
- Test generation agent will skip trivial code (getters, simple pass-throughs)
- Test output parsing tolerates some variation but warns on unparseable tests
```

**Step 6: Commit**

```bash
git add plugins/my-personal-tools/commands/fix-loop.md
git commit -m "feat: add test generation orchestration to fix-loop command"
```

---

## Task 7: Create test-generation-guide.md for external plugin developers

**Files:**
- Create: `plugins/my-personal-tools/docs/TEST_GENERATION_GUIDE.md`

**Step 1: Write the guide**

```markdown
# Test Generation Guide for Validator Developers

When your validator is integrated with fix-loop's test generation system, you can provide context that helps the test generator write better tests.

## Overview

After your validator fixes code, the test-generator agent runs. It uses:
1. Git diff (what changed)
2. **Validator context** (why it changed) ← You control this
3. Project patterns (how to test)

This guide explains how to provide clear context so tests are meaningful.

## Providing Validator Context

When your validator output includes fixes, add context that explains:
- **What design pattern or logic was introduced** - "Extracted domain validation logic"
- **What public APIs were created** - "New UserRegistrationService class with registerUser() method"
- **What business logic matters** - "Email validation must run before user creation"

### Example: DDD/OOP Validator

Your validator might fix anemic models by extracting business logic:

```
🔴 CRITICAL: Anemic domain model - User has no behavior
   📍 src/domain/User.ts:8
   💡 Add validateEmail method to User entity

CONTEXT FOR TEST GENERATION:
- Pattern: DDD domain model enrichment
- New public methods: validateEmail()
- Business logic: Email must contain @ symbol
- Test focus: Validation rules and error cases
```

The test generator will then create:

```typescript
describe('User', () => {
  describe('validateEmail', () => {
    it('should accept valid email formats', () => { ... });
    it('should reject invalid email formats', () => { ... });
  });
});
```

### Example: DRY Validator

Your validator might extract duplicated constants:

```
🟠 HIGH: Magic string duplicated 3 times
   📍 src/config.ts, src/utils.ts, src/api.ts (all use "USER_ROLE")
   💡 Create USER_ROLE constant in constants.ts, import everywhere

CONTEXT FOR TEST GENERATION:
- Pattern: DRY constant extraction
- New constant: USER_ROLE in src/constants.ts
- Usage locations: config.ts, utils.ts, api.ts
- Test focus: Constant is used correctly, has expected value
```

Test generator will verify the constant works correctly.

## Output Format for Validator Context

Include context in your validator output like this:

```
🔴 CRITICAL: [Issue]
   📍 file:line
   💡 [Recommendation]

   TEST_CONTEXT:
   - Pattern: [What you introduced: "DDD service", "Extracted constant", "Mapper", etc.]
   - Creates: [What new code was created: classes, methods, functions]
   - Public API: [What's publicly callable: method names, class names]
   - Business Logic: [What matters to test: validation rules, behavior]
   - Test Focus: [What tests should verify: happy paths, errors, side effects]
```

### Full Example

```
🔴 CRITICAL: User registration has no business logic validation
   📍 src/services/UserService.ts:12
   💡 Create UserRegistrationService with validateEmail and checkDuplicate methods

   TEST_CONTEXT:
   - Pattern: DDD service extraction
   - Creates: UserRegistrationService class
   - Public API: registerUser(id, email), validateEmail(email), checkDuplicate(email)
   - Business Logic:
     * Email validation: must contain @
     * Duplicate check: query repository before creating
     * Creation: only after both validations pass
   - Test Focus: Happy path registration, validation errors, duplicate detection
```

## When Context is Optional

You don't always need context. The test generator will attempt to understand code from:
- Git diff alone
- Method/class names
- Existing project test patterns

Context is most helpful for:
- Complex business logic that intent isn't obvious
- Multiple related changes that form one feature
- Non-standard patterns that need explanation
- Error cases that should be tested

## Best Practices

1. **Be specific** - "DDD service" is good, but "Service with domain validation logic" is better
2. **List all public methods** - So tests know what to call
3. **Highlight business rules** - "Email must contain @, be <256 chars"
4. **Mention dependencies** - "Uses UserRepository for duplicate checking"
5. **Keep it concise** - 3-5 bullets per fix, not paragraphs

## Example: Complete Validator Output

```
🔴 CRITICAL: User entity is anemic (data only, no behavior)
   📍 src/domain/User.ts:1
   💡 Move email validation logic into User entity as private validateEmail() method

   TEST_CONTEXT:
   - Pattern: DDD - move validation behavior to entity
   - Creates: User class with private validateEmail(email) method
   - Public API: constructor(id, email) - should validate on creation
   - Business Logic: Email format validation (must contain @)
   - Test Focus: Constructor validation, error on invalid email, success on valid email

🟠 HIGH: User service has no duplicate checking
   📍 src/services/UserService.ts:8
   💡 Extract duplicate checking into UserRegistrationService.registerUser()

   TEST_CONTEXT:
   - Pattern: Service extraction with dependency injection
   - Creates: UserRegistrationService(userRepository) class with registerUser method
   - Public API: registerUser(id, email) -> Promise<User>
   - Business Logic: Check repository for existing user, throw error if found, create if not
   - Dependencies: Depends on UserRepository interface (mock in tests)
   - Test Focus: Happy path creation, error on duplicate, repository interaction
```

## Integration with Test Generator

The test generator will:
1. Parse your context from the fixing findings
2. Understand the design pattern you introduced
3. Generate tests aligned with your stated business logic
4. Create tests for stated public API
5. Focus on areas you marked as important

The more specific your context, the better the tests will be.

## Questions?

If test generator creates tests that don't match your intent:
1. Review your context - was it clear?
2. Make context more specific or complete
3. You can also manually adjust generated tests if needed

Remember: generated tests are a starting point. They can always be refined manually.
```

**Step 2: Verify the file**

Run:
```bash
head -20 /Users/markky21/Projects/claude-skills/plugins/my-personal-tools/docs/TEST_GENERATION_GUIDE.md
```

Expected: File exists with markdown headers

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/docs/TEST_GENERATION_GUIDE.md
git commit -m "docs: add test generation guide for validator developers"
```

---

## Task 8: Update README with test-sealed fix-loop information

**Files:**
- Modify: `plugins/my-personal-tools/README.md` - add test generation section

**Step 1: Read the current README**

Read the file to understand structure.

**Step 2: Add section about test generation**

In the README, add a new section explaining the test feature. Find the "Fix Loop" section (if exists) or add near end:

```markdown
### Fix-Loop with Test Generation

The `/fix-loop` command now includes optional test generation:

- **Automatic test creation** - After validators fix code, tests are automatically generated for new/modified code
- **Test-driven iteration** - If tests fail, validators re-analyze with test failure context and iterate
- **Validation & regression protection** - Tests ensure fixes work correctly and prevent regressions

**Using test generation:**

```bash
/fix-loop
```

When prompted, select "Yes" for test generation (default). Choose test scope and coverage level.

**Disabling tests (advanced):**

```bash
/fix-loop --skip-tests
```

Tests are **enabled by default** but can be disabled for faster iteration if needed.

**How it works:**

1. Validators apply code quality fixes
2. Test generator analyzes changes and creates appropriate tests
3. Tests run - if all pass, done
4. If tests fail, failures are analyzed and fed back to validators
5. Validators fix implementation based on test expectations
6. Loop continues until tests pass or max iterations reached

This ensures fixes are not just "correct" by quality metrics, but actually **work** as intended.
```

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/README.md
git commit -m "docs: add test-sealed fix-loop documentation to README"
```

---

## Task 9: Verify all files are in place and correct

**Files:**
- Verify all new and modified files exist and have correct content

**Step 1: Check all new agent files exist**

Run:
```bash
for file in fix-loop-test-generator fix-loop-test-runner fix-loop-failure-analyzer; do
  if [ -f "plugins/my-personal-tools/agents/${file}.md" ]; then
    echo "✅ $file exists"
  else
    echo "❌ $file missing"
  fi
done
```

Expected: All three files exist

**Step 2: Check testing-patterns.json exists**

Run:
```bash
[ -f "plugins/my-personal-tools/testing-patterns.json" ] && echo "✅ testing-patterns.json exists" || echo "❌ testing-patterns.json missing"
```

Expected: `✅ testing-patterns.json exists`

**Step 3: Check test guide exists**

Run:
```bash
[ -f "plugins/my-personal-tools/docs/TEST_GENERATION_GUIDE.md" ] && echo "✅ TEST_GENERATION_GUIDE.md exists" || echo "❌ TEST_GENERATION_GUIDE.md missing"
```

Expected: `✅ TEST_GENERATION_GUIDE.md exists`

**Step 4: Verify JSON is valid**

Run:
```bash
cat plugins/my-personal-tools/testing-patterns.json | jq . > /dev/null && echo "✅ JSON valid" || echo "❌ JSON invalid"
```

Expected: `✅ JSON valid`

**Step 5: Check git status**

Run:
```bash
git status --short | grep -E "^\s*[AM]" | wc -l
```

Expected: Number of added/modified files matches our implementation

**Step 6: Final summary commit**

Run:
```bash
git log --oneline -10
```

Expected: Should show all our commits from previous tasks

**Step 7: Summary**

All components are in place:
- ✅ 3 new agents created (test-generator, test-runner, failure-analyzer)
- ✅ testing-patterns.json manifest created
- ✅ fix-loop command orchestration updated
- ✅ fix-loop-configurator prompts expanded
- ✅ TEST_GENERATION_GUIDE.md created
- ✅ README updated
- ✅ All changes committed to git

---

## Post-Implementation Checklist

When execution is complete, verify:

- [ ] All 3 new agents are syntactically correct and loadable
- [ ] testing-patterns.json is valid JSON with all required fields
- [ ] fix-loop command has test orchestration logic in all phases
- [ ] fix-loop-configurator includes test generation prompts
- [ ] test-generation-guide.md is clear and helpful
- [ ] All changes are committed with descriptive messages
- [ ] No uncommitted changes remain

---

## Summary

This implementation plan adds comprehensive test generation capabilities to fix-loop:

**What gets built:**
- Test generator agent (analyzes changes, generates appropriate tests)
- Test runner agent (executes tests, reports results)
- Failure analyzer agent (interprets test failures, guides fixes)
- Interactive configurator prompts (enable/configure test generation)
- Orchestration logic (manage full test-driven fixing loop)

**How it works:**
1. User runs `/fix-loop` and selects "Yes" for test generation
2. Validators apply code quality fixes
3. Test generator creates tests for the changes
4. Tests run - if pass, done; if fail, analyzed and fed back to validators
5. Validators iterate until tests pass or max iterations reached

**Key benefits:**
- Ensures fixes actually work, not just "correct" by metrics
- Provides regression protection through automated testing
- Uses project's own testing patterns and conventions
- Senior-level test thinking (no trivial tests, focus on business logic)
- Fully integrated into existing fix-loop workflow
