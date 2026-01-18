# Test-Sealed Fix-Loop Design

**Date:** 2026-01-18
**Status:** Approved
**Scope:** Enhance fix-loop to automatically generate tests for fixes and iterate until tests pass

## Overview

Extend fix-loop with test-driven validation: after validators apply fixes, automatically generate tests for the new/modified code, run them, and feed test failures back to validators for iterative fixing until all tests pass.

## Problem Statement

Currently, fix-loop applies code quality fixes but doesn't validate that fixes actually work or provide regression protection. Generated fixes could be incomplete or break at runtime. Tests would catch these issues and guide further fixes.

## Solution Overview

Implement a **test-driven fixing loop**:
1. Validators apply fixes
2. Test generator creates tests for new/modified code (analyzing git diff, validator context, project patterns)
3. Tests run - if pass, done; if fail, feed failures to validators
4. Validators re-analyze with test failure context and apply fixes
5. Loop until tests pass or max iterations hit

## Section 1: Enhanced Fix-Loop Flow with Test Generation

**Phase 1: Fix & Test Setup**
- Run selected validators to apply fixes
- If `generate tests = yes`, invoke the test-generator agent
- Test generator analyzes: git diff (what changed), validator output (why it changed), existing test patterns (how to test)
- Generate tests in the same directory structure and style as the project

**Phase 2: Test Execution**
- Run the generated tests
- If all pass → Success, fix-loop completes
- If tests fail → Move to Phase 3

**Phase 3: Test-Driven Fixing**
- Pass test failure output + implementation code + validator context back to fix-loop validators
- Validators re-analyze with new context: "tests expect X, but implementation does Y"
- Apply fixes to make tests pass
- Re-run tests
- Loop until tests pass OR max iterations reached

**Integration**: Users toggle test generation in the existing questionnaire. Default is enabled. If disabled, fix-loop works as it does today.

## Section 2: Test Generator Agent Architecture

The **test-generator agent** is a new specialized agent that creates tests intelligently by analyzing three data sources:

**Input 1: Git Diff Analysis**
- Scan what changed: new methods, services, classes, mappers, refactored code
- Extract signature information: method names, parameters, return types
- Identify the "units" to test (a service, a mapper, a method)

**Input 2: Validator Context**
- Validators provide: why they changed the code, what design pattern was introduced, what business logic was added
- Example: "Introduced DomainService for user registration with business logic validation"
- This context helps generator understand *intent*, not just syntax

**Input 3: Project Testing Patterns**
- Scan existing test files to learn:
  - Where tests live (`__tests__`, `.test.ts`, `spec/` folders?)
  - Naming conventions (`UserService.test.ts` or `user.service.spec.ts`?)
  - Test structure (Jest, Vitest, Mocha? Describe/it blocks? AAA pattern?)
  - Assertion styles (expect, should, assert?)
  - Fixture patterns (factories, mocks, builders?)
  - Import/setup patterns

**Output: Generated Tests**
- Write tests for each new/modified code unit
- Place in locations that match project patterns
- Use naming and assertion styles from project
- Cover: happy paths, edge cases, integration with dependencies
- Include clear comments explaining what's being tested and why

**Test Quality Heuristics** (senior-level thinking):
- Don't test trivial getters/setters
- Test business logic, validation, side effects
- Mock external dependencies appropriately
- Use realistic test data
- Test error conditions

## Section 3: Test Execution & Failure-Driven Fixing

**Test Execution Phase**
- Run generated tests in the project's test environment
- Capture output: test names, pass/fail status, error messages, stack traces
- Categorize failures:
  - **Assertion failures** - Test expected X, got Y (implementation incomplete or wrong)
  - **Runtime errors** - Code crashed (missing logic, type errors)
  - **Setup failures** - Tests couldn't run (import errors, missing fixtures)

**Feeding Failures Back to Validators**
- Create a failure report: which tests failed, what they expected, what the implementation did
- Include: the test code, the implementation code, the validator's original fix rationale
- Pass to the **same validators that made the original fixes**
- Validators now have new context: "This DDD service method is missing validation logic that tests expect"

**Validator Re-Analysis & Fixes**
- Validators understand: the original design intent + what tests reveal is missing
- Apply targeted fixes: add missing validation, complete partial implementations, fix type mismatches
- Generate explanation of what was fixed

**Loop Control**
- After fixes applied: re-run tests
- If all pass → Success, loop terminates
- If tests still fail: loop back to validators (up to max iterations)
- If iteration count hit → Report: "Reached max iterations with N tests still failing. Manual review needed."

**Error Handling**
- If validator can't fix a particular issue → skip that validator for next iteration
- If same test fails identically twice in a row → likely a deeper issue, consider stopping

## Section 4: Questionnaire Integration & Configuration

**Updated Questionnaire Flow**

```
═══════════════════════════════════════════════════════════════
 Fix-Loop Configurator
═══════════════════════════════════════════════════════════════

📋 SELECT VALIDATORS
─────────────────────────────────────────────────────────────
[User selects built-in and external validators as before]

🎯 SEVERITY FILTER
─────────────────────────────────────────────────────────────
[Existing severity selection: CRITICAL, HIGH, MEDIUM, LOW]

⚡ MAX ITERATIONS
─────────────────────────────────────────────────────────────
[Existing iteration count: 3, 5 (recommended), 10]

🧪 GENERATE & RUN TESTS (NEW)
─────────────────────────────────────────────────────────────
✓ Yes - Generate tests for fixes, run them, iterate if failures
☐ No - Apply fixes only, skip test generation
```

**Test Generation Options**

When test generation is enabled (`Yes`), show optional advanced config:

```
   Test Scope:
   ✓ All new/modified code - Test every change (Recommended)
   ☐ Only business logic - Skip utilities and helpers
   ☐ Only failing code - Test only code that validators flagged

   Test Coverage:
   ✓ Happy paths + edge cases
   ☐ Happy paths only
   ☐ Comprehensive (include error scenarios)
```

**Default Behavior**
- Test generation: **Enabled by default** (✓ Yes)
- Test scope: **All new/modified code** (most thorough)
- Test coverage: **Happy paths + edge cases** (senior-level)

**CLI Support** (Optional, for automation)
```bash
/fix-loop --with-tests
/fix-loop --with-tests --test-scope business-logic
/fix-loop --skip-tests
```

## Section 5: Implementation Architecture & Components

**New Agents Created**

1. **fix-loop-test-generator** (NEW)
   - Analyzes git diff, validator context, project test patterns
   - Generates test files with appropriate content
   - Places tests in project-correct locations with project-correct style
   - Outputs: list of generated test files and their content

2. **fix-loop-test-runner** (NEW)
   - Executes tests in the project's test environment (Jest, Vitest, Mocha, etc.)
   - Parses test output into structured format: which tests passed/failed, error details
   - Outputs: test results in standardized format for parsing

3. **fix-loop-failure-analyzer** (NEW)
   - Receives test failures, the implementation code, and validator context
   - Extracts what tests expected vs what implementation does
   - Formats failure report clearly for validators
   - Outputs: structured failure context to feed back to validators

**Modified Components**

1. **fix-loop.md command** (MODIFY)
   - Add test generation question to questionnaire
   - Orchestrate the new flow: fixes → test generation → test execution → failure handling
   - Add loop logic: if tests fail, re-invoke validators with failure context
   - Track test status across iterations

2. **fix-loop-fixer agent** (MODIFY - minimal)
   - Now receives both code quality issues AND test failure context
   - Uses failure context as additional signal when applying fixes
   - No major changes, just new input source

3. **Existing validators** (NO CHANGE)
   - Work as-is, just receive new context when fixing test failures
   - Output format unchanged

**Data Flow**

```
/fix-loop command (orchestrator)
    ↓
[Existing] Apply validator fixes
    ↓
[NEW] Test generator analyzes changes + patterns
    ↓
[NEW] Write test files
    ↓
[NEW] Run tests
    ├─→ Tests pass: Done
    └─→ Tests fail:
        ↓
    [NEW] Analyze failures
        ↓
    [Existing] Validators re-fix with failure context
        ↓
    Loop until pass or max iterations
```

## Section 6: File Structure & Project Organization

**Plugin Structure**

```
plugins/my-personal-tools/
├── validators.json (existing - built-in validator manifest)
├── testing-patterns.json (NEW - reference patterns for test generation)
├── commands/
│   └── fix-loop.md (MODIFIED - enhanced questionnaire + orchestration)
├── agents/
│   ├── fix-loop-validator-discovery.md (existing)
│   ├── fix-loop-configurator.md (existing)
│   ├── fix-loop-fixer.md (existing)
│   ├── fix-loop-test-generator.md (NEW)
│   ├── fix-loop-test-runner.md (NEW)
│   └── fix-loop-failure-analyzer.md (NEW)
├── skills/
│   └── fix-loop/
│       └── SKILL.md (MODIFIED - orchestration logic)
└── docs/
    └── TEST_GENERATION_GUIDE.md (NEW - for external plugin developers)
```

**New File: testing-patterns.json**

This file helps the test generator understand the project's testing conventions.

**New File: test-generation-guide.md**

Documentation for external plugin developers on how their validators can provide test context.

## Section 7: Success Criteria & Completion Metrics

**Core Functionality**
1. ✅ Test generation question appears in fix-loop questionnaire with "Yes" as default
2. ✅ When enabled, test-generator agent analyzes git diff + validator context + project patterns
3. ✅ Generated tests are placed in project-correct locations (matching existing test structure)
4. ✅ Generated tests use project-correct syntax and patterns (matching assertion style, naming conventions)
5. ✅ Tests are executed automatically after generation
6. ✅ Test results are parsed and analyzed

**Loop Control**
7. ✅ If tests pass on first run: fix-loop completes successfully with test summary
8. ✅ If tests fail: failure context is fed back to validators with clear error messages
9. ✅ Validators receive test failures + implementation code + original fix rationale
10. ✅ Validators apply fixes and tests are re-run
11. ✅ Loop continues until tests pass OR max iterations reached
12. ✅ Loop termination reasons are clearly reported (success, max iterations, stall detection)

**Quality Metrics**
13. ✅ Generated tests have meaningful names and comments (not generic/useless)
14. ✅ Tests avoid testing trivial code (getters/setters, simple passes-through)
15. ✅ Tests focus on business logic and side effects (senior-level thinking)
16. ✅ Error handling: when validators can't fix a test failure, the system handles gracefully

**User Experience**
17. ✅ Full flow works with `/fix-loop` command with no special flags needed
18. ✅ Users can opt-out with `/fix-loop --skip-tests`
19. ✅ Clear reporting at each phase: fixes applied → tests generated → test results → fixes applied...
20. ✅ Error messages are helpful when tests fail or generation fails

## Glossary

- **Test Generator**: Agent that analyzes changes and creates test files
- **Test Runner**: Agent that executes tests and reports results
- **Failure Analyzer**: Agent that interprets test failures and formats context for validators
- **Test-Driven Fixing**: Process of iterating validator fixes until tests pass
- **Validator Context**: Information from validators about why they made changes
- **Project Patterns**: Detected testing conventions and styles from existing test files
