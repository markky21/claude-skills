# Fix-Loop Dynamic Validator Discovery Design

**Date:** 2026-01-18
**Status:** Approved
**Scope:** Enhanced fix-loop command to dynamically discover and integrate built-in and external validators

## Overview

The enhanced fix-loop command enables users to select from all available validators—both built-in validators in the my-personal-tools plugin and validators from external plugins installed in their project. The system uses standardized metadata and output formats to allow seamless integration of validators across multiple sources.

## Problem Statement

Currently, fix-loop has hardcoded validators:
- ddd-oop-validator
- dry-violations-detector
- clean-code-validator
- react-nextjs-validator

Users cannot:
1. See what other validators are available in external plugins
2. Easily toggle external validators like `react-best-practices-validator` from an external plugin
3. Mix built-in and external validators in a single fix-loop run

## Solution Overview

Implement three integrated components:

1. **Validator Discovery** - Scan built-in and external plugins for validator metadata
2. **Interactive Configurator** - Let users select validators, grouped by source and category
3. **Standardized Output Format** - Require all validators to produce findings in a consistent format

## Section 1: Architecture Overview

The enhanced fix-loop has three core phases:

### Phase 1: Validator Discovery

When fix-loop starts, it scans two sources:

**Built-in validators** in my-personal-tools plugin:
- ddd-oop-validator
- dry-violations-detector
- clean-code-validator
- react-nextjs-validator

**External plugin validators** by reading installed plugins' `plugin.json` files for a `validators` metadata array.

Discovery process:
1. Read the current project's installed plugins from Claude Code's plugin registry
2. For each plugin, check if it has a `validators` array in plugin.json
3. Fallback: scan for agents named `*-validator` (catches existing validators that haven't updated their manifests)
4. Validate that declared agents actually exist (warn if not found)
5. Build a consolidated list of all available validators with their metadata

### Phase 2: Configurator

Users see an interactive menu grouping validators by source and category. They can toggle which validators to run and set severity filters. This preference gets passed to Phase 3.

### Phase 3: Execution

The fix-loop command launches selected validators in parallel, parses their standardized output, applies fixes, and iterates until completion or termination condition.

**Architecture benefits:**
- Concerns are separated: discovery is independent, configurator doesn't know about execution details, validators just conform to one output format
- Easy to extend: new validators in external plugins are automatically discovered
- User control: users choose which validators to run each time

## Section 2: Validator Metadata Specification

### External Plugin Metadata

External plugins declare their validators using a standardized metadata structure in `plugin.json`:

```json
{
  "name": "react-best-practices",
  "version": "1.0.0",
  "validators": [
    {
      "agentName": "react-best-practices-validator",
      "displayName": "React Best Practices",
      "description": "Validates React 19+ patterns, hook selection, and performance optimizations",
      "category": "react",
      "version": "1.0.0"
    },
    {
      "agentName": "next-patterns-validator",
      "displayName": "Next.js Patterns",
      "description": "Validates Next.js 15+ app router patterns and server/client boundaries",
      "category": "nextjs",
      "version": "1.0.0"
    }
  ]
}
```

**Metadata fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `agentName` | Yes | Internal agent name, must match actual agent definition |
| `displayName` | Yes | Human-friendly name shown in configurator |
| `description` | Yes | What this validator checks (1-2 sentences) |
| `category` | No | Group validators: "react", "nextjs", "ddd", "dry", "performance", etc. |
| `version` | No | Validator version for tracking compatibility |

### Built-in Validator Manifest

Built-in validators are defined in `validators.json` in the my-personal-tools plugin:

```json
{
  "validators": [
    {
      "agentName": "ddd-oop-validator",
      "displayName": "DDD/OOP Validator",
      "description": "Validates Domain-Driven Design and OOP compliance",
      "category": "architecture"
    },
    {
      "agentName": "dry-violations-detector",
      "displayName": "DRY Violations Detector",
      "description": "Finds duplicated code, constants, and magic values",
      "category": "dry"
    },
    {
      "agentName": "clean-code-validator",
      "displayName": "Clean Code Validator",
      "description": "Validates naming, function size, and SOLID principles",
      "category": "cleancode"
    },
    {
      "agentName": "react-nextjs-validator",
      "displayName": "React/Next.js Validator",
      "description": "Validates React 19+ and Next.js 15+ patterns",
      "category": "react"
    }
  ]
}
```

## Section 3: Configurator Discovery & UI

### Discovery Process

When `/fix-loop` is invoked without arguments:

1. Scan installed plugins directory for `plugin.json` files
2. For each plugin, extract `validators` metadata array
3. Fallback: scan for agents named `*-validator` if no metadata found
4. Validate that declared agents actually exist (warn if not found)
5. Merge with built-in validators from `validators.json`
6. Group all validators by category and source

### UI Layout

```
═══════════════════════════════════════════════════════════════
 Fix-Loop Configurator
═══════════════════════════════════════════════════════════════

📋 SELECT VALIDATORS (choose at least one)
─────────────────────────────────────────────────────────────

Built-in Validators [my-personal-tools]
  ✓ DDD/OOP Validator (architecture)
    Validates Domain-Driven Design and OOP compliance

  ✓ DRY Violations Detector (dry)
    Finds duplicated code, constants, and magic values

  ✓ Clean Code Validator (cleancode)
    Validates naming, function size, and SOLID principles

  ✓ React/Next.js Validator (react)
    Validates React 19+ and Next.js 15+ patterns

External Plugins
  react-best-practices [1.2.0]
    ☐ React Best Practices Validator (react)
      Validates React 19+ patterns, hook selection, and performance optimizations

    ☐ React Testing Library Validator (testing)
      Checks React Testing Library best practices and patterns

  ddd-patterns-lib [2.0.1]
    ☐ Advanced DDD Patterns Validator (architecture)
      Validates advanced DDD patterns: bounded contexts, aggregates, events
```

### User Interaction

1. Users check/uncheck individual validators
2. Continue to next step when ready
3. Default: Built-in validators pre-selected, external validators unchecked

### Severity Filter (Existing)

```
🎯 SEVERITY FILTER
─────────────────────────────────────────────────────────────
✓ CRITICAL - Must fix, blocks merge
✓ HIGH - Should fix, high risk
☐ MEDIUM - Address soon
☐ LOW - Nice to have
```

### Iteration Count (Existing)

```
⚡ MAX ITERATIONS
─────────────────────────────────────────────────────────────
☐ 3 - Quick pass
✓ 5 - Balanced (Recommended)
☐ 10 - Thorough
```

The configurator stores user choices and passes them to the validation engine.

## Section 4: Validator Output Specification

All validators (built-in and external) must conform to a standardized output format. This enables fix-loop to reliably parse findings, count issues, and track progress across iterations.

### Required Output Format

```
🔴 CRITICAL: [Concise description of the issue]
   📍 file/path.ts:42
   💡 [Specific recommendation for fixing this issue]

🟠 HIGH: [Issue description]
   📍 file/path.ts:87
   💡 [Recommendation]

🟡 MEDIUM: [Issue description]
   📍 file/path.ts:120
   💡 [Recommendation]

🟢 LOW: [Issue description]
   📍 file/path.ts:155
   💡 [Recommendation]
```

### Specification Details

- **Severity emoji** (required): 🔴 CRITICAL, 🟠 HIGH, 🟡 MEDIUM, 🟢 LOW
- **Colon after severity**: `: ` (space after colon)
- **Issue description** (required): 1 sentence, clear and actionable
- **Location line** (required): `📍 file/path.ts:42` format exactly
- **Recommendation line** (required): `💡 [1-2 sentence fix instruction]`
- **Blank lines** between findings for readability

### Parsing Rules

Fix-loop uses regex patterns to extract findings:
- Severity + description: `/(🔴|🟠|🟡|🟢)\s+([A-Z]+):\s+(.+)/`
- Location: `/📍\s+(.+):(\d+)/`
- Recommendation: `/💡\s+(.+)/`
- Any findings not matching this pattern are ignored with a warning

### Example Output

From react-nextjs-validator adapted to spec:

```
🔴 CRITICAL: Hook called conditionally inside if statement
   📍 src/components/Form.tsx:42
   💡 Move useState call outside the conditional block. Hooks must be called unconditionally on every render.

🟠 HIGH: Multiple useState calls managing related state
   📍 src/hooks/useFormState.ts:15
   💡 Consider using useReducer instead of 5 related useState calls for better state management.

🟡 MEDIUM: Derived state stored in component state
   📍 src/components/UserProfile.tsx:28
   💡 Calculate filteredUsers directly instead of storing in state. Use useMemo if computation is expensive.
```

### Validation

- Before launching fix-loop, all selected validators are checked for compliance
- External validators that don't conform are flagged with error: "Validator X outputs non-standard format. Please update."
- Built-in validators are guaranteed compliant

## Section 5: Integration with Fix-Loop

### End-to-End Flow

**Step 1: Validator Discovery & Loading**
```
user runs: /fix-loop
  ↓
discover built-in validators from validators.json
discover external validators from installed plugins/plugin.json
validate that all declared agents exist
load metadata (displayName, category, description)
  ↓
if discovery errors found, warn user but continue
```

**Step 2: Interactive Configurator (if no CLI args)**
```
show validator selection menu (grouped by source, then category)
show severity filter options
show iteration count options
  ↓
user selects validators, severity levels, iterations
store selection in variables
```

**Step 3: Validation Loop (per iteration)**
```
for each selected validator:
  launch Task with agent name and standardized prompt
  pass: branch diff, changed files, validator type
  receive: output in standard format

collect all outputs from parallel validators
parse with regex to extract findings (severity, location, recommendation)
filter by selected severity levels
count issues: currentIssueCount = matching findings

if currentIssueCount == 0 → Success, stop
if iteration >= maxIterations → Max reached, stop
if currentIssueCount >= previousIssueCount for 2 iterations → Stall, stop

if continuing:
  aggregate all findings
  pass to fix-loop-fixer agent
  track total fixes applied
  loop to next iteration
```

**Step 4: Reporting**
```
display summary:
  - iterations used
  - total issues fixed
  - remaining issues (if any)
  - which validators were run
  - which validators found issues
```

### Key Changes to Existing Fix-Loop Command

1. **Before Step 1**: Add validator discovery code
2. **Before Step 2**: Modify AskUserQuestion to dynamically build validator options from discovered list
3. **During Step 3b**: Make validator agent names dynamic (from metadata instead of hardcoded)
4. **During Step 3c**: Parsing logic stays the same but now handles arbitrary validators
5. **New validation**: Before launching validators, warn if any don't match output spec

### CLI Argument Support (Optional, Advanced)

```bash
/fix-loop --validators react-best-practices,ddd-oop-validator --severity critical,high --iterations 5
```

Skips configurator and runs directly with specified validators.

### Error Handling

- If external plugin disappears between discovery and execution → skip with warning
- If validator produces non-standard output → attempt parsing, log unparseable lines, continue
- If validator agent fails to run → record as validation error, skip that validator for iteration

## Implementation Priorities

### Phase 1: Core Discovery & Configurator
- Implement validator discovery (built-in + external)
- Build dynamic configurator UI
- Update fix-loop command to use discovered validators

### Phase 2: Output Validation & Parsing
- Add output format validation before running validators
- Update parsing logic to handle external validators
- Add error messages for non-compliant output

### Phase 3: Polish & Documentation
- Document validator metadata spec for external plugin developers
- Create examples of compliant validators
- Write migration guide for existing validators

## Success Criteria

1. ✅ Built-in validators appear in configurator and can be toggled
2. ✅ External plugin validators appear grouped by plugin
3. ✅ Users can select any combination of validators
4. ✅ All validators produce standardized output
5. ✅ Fix-loop correctly parses findings from all validator types
6. ✅ Iterative fixing works with mixed built-in and external validators
7. ✅ External plugin developers can add validators by updating plugin.json

## File Structure

```
plugins/my-personal-tools/
├── validators.json (NEW - built-in validator manifest)
├── commands/
│   └── fix-loop.md (MODIFIED - enhanced command with discovery)
├── agents/
│   ├── fix-loop-validator-discovery.md (NEW - discovery agent)
│   ├── fix-loop-configurator.md (NEW - configurator agent)
│   └── fix-loop-fixer.md (EXISTING - no changes needed)
└── skills/
    └── fix-loop/
        └── SKILL.md (MODIFIED - updated to use dynamic validators)
```

## Glossary

- **Validator**: A code review agent that checks for specific types of issues (DDD, DRY, React patterns, etc.)
- **Finding**: A specific issue identified by a validator, with severity, location, and recommendation
- **Metadata**: Structural information about a validator (name, description, category)
- **External Plugin**: A Claude Code plugin installed in the user's project from an external source
- **Built-in Validator**: A validator defined in my-personal-tools plugin
- **Configurator**: Interactive menu where users select validators and options
- **Output Spec**: Standardized format all validators must use when reporting findings
