---
description: Run comprehensive multi-agent code review on current branch
allowed-tools: Bash(git *), Task
---

# Branch Review Command

## Context

- Current branch: !`git branch --show-current`
- Git status: !`git status --short`
- Files changed vs main: !`git diff main...HEAD --stat 2>/dev/null || git diff HEAD~5...HEAD --stat`
- Recent commits: !`git log --oneline -5`

## Your Task

Execute a comprehensive code review of the current branch using parallel agents.

### Step 1: Analyze Scope

Based on the context above, determine:
- How many files changed?
- What type of change? (feature/fix/refactor/test)
- Which modules/domains are affected?

### Step 2: Launch Parallel Review Agents

Use the Task tool to launch these agents **in parallel** (they're independent):

**Agent 1: DDD/OOP Validator**
```
Review the branch diff for DDD/OOP compliance:
- Check for anemic domain models (data containers without behavior)
- Find Tell/Don't Ask violations (extracting data to calculate in services)
- Identify parameter bloat (services with many primitive parameters)
- Look for defensive null checks on invariant data
- Verify methods are on objects that own the data

Use: git diff main...HEAD (or recent commits if no main)
Reference the ddd-oop skill patterns.

Report findings with severity (CRITICAL/HIGH/MEDIUM/LOW), file:line, and recommended fix.
```

**Agent 2: DRY Violations Detector**
```
Review the branch diff for DRY violations:
- Find copy-pasted code blocks
- Identify magic strings/numbers that should be constants/enums
- Spot similar patterns that could be unified
- Look for reimplemented helpers/utilities

Use: git diff main...HEAD (or recent commits if no main)
Reference the dry-violations skill patterns.

Report duplications with locations, severity, and extraction suggestions.
```

**Agent 3: Architecture Compliance**
```
Review the branch diff for architecture compliance:
- Verify layer separation (domain/application/infrastructure)
- Check ORM/framework decorators are only in infrastructure
- Verify mappers exist between layers if needed
- Check repository pattern usage
- Verify dependency direction (inward toward domain)

Use: git diff main...HEAD --name-only to see file locations
Use: git diff main...HEAD to see actual changes

Report violations with file:line and recommended structure.
```

**Agent 4: Test Coverage**
```
Review the branch for test coverage:
- List changed source files
- Check if each has corresponding test file
- Assess test quality if tests exist
- Identify missing test coverage

Use: git diff main...HEAD --name-only
Compare source files to test files.

Report coverage percentage, missing tests, and quality observations.
```

**Agent 5: Clean Code Validator**
```
Review the branch diff for clean code principles:
- Check function/method naming (intention-revealing?)
- Check function size (<20 lines ideal)
- Look for code smells (long parameter lists, feature envy, god classes)
- Check SOLID principle adherence
- Verify error handling patterns

Use: git diff main...HEAD

Report issues with severity, principle violated, and refactoring suggestions.
```

### Step 3: Generate Summary Report

After all agents complete, compile findings into this format:

```markdown
# Code Review Summary

## Branch: [name]
## Changes: [X files, Y additions, Z deletions]

---

## 🏗️ Architecture: [✅ PASS / 🟡 NEEDS WORK / 🔴 FAIL]
[Key findings]

## 🎯 DDD/OOP: [✅ PASS / 🟡 NEEDS WORK / 🔴 FAIL]
[Key findings]

## 🚫 DRY: [✅ PASS / 🟡 NEEDS WORK / 🔴 FAIL]
[Key findings]

## 🧹 Clean Code: [✅ PASS / 🟡 NEEDS WORK / 🔴 FAIL]
[Key findings]

## ✅ Tests: [✅ PASS / 🟡 NEEDS WORK / 🔴 FAIL]
[Key findings]

---

## 🚦 Quality Gates

| Gate | Status |
|------|--------|
| Critical Issues | [count] |
| High Issues | [count] |
| Ready to Merge | [Yes/No] |

---

## 📋 Action Items

1. [Priority fixes]
2. [Recommendations]
```

### Severity Guide

- 🔴 **CRITICAL**: Must fix before merge
- 🟠 **HIGH**: Should fix before merge
- 🟡 **MEDIUM**: Address soon (can be follow-up)
- 🟢 **LOW**: Nice to have

### Important

- Run all 5 agents **in parallel** using multiple Task tool calls
- Each agent should reference the appropriate skill (ddd-oop, dry-violations)
- Focus on actual patterns found, not hypothetical issues
- Include specific file:line references
- Highlight positives too - what was done well?
