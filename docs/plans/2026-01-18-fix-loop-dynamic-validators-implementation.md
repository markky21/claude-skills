# Fix-Loop Dynamic Validator Discovery Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement dynamic validator discovery for fix-loop to allow users to select from built-in and external plugin validators through an interactive configurator.

**Architecture:** Three-phase implementation building from foundation (validator manifest & discovery) through integration (updated fix-loop command) to polish (validation & error handling).

**Tech Stack:** Claude Code command/agent system, JSON metadata, regex parsing, AskUserQuestion for interactive UI

---

## Phase 1: Core Discovery & Configurator Foundation

### Task 1: Create Built-in Validator Manifest

**Files:**
- Create: `plugins/my-personal-tools/validators.json`

**Step 1: Write validators.json**

```json
{
  "validators": [
    {
      "agentName": "ddd-oop-validator",
      "displayName": "DDD/OOP Validator",
      "description": "Validates Domain-Driven Design and OOP compliance - detects anemic models, Tell/Don't Ask violations, method placement issues",
      "category": "architecture",
      "version": "1.0.0"
    },
    {
      "agentName": "dry-violations-detector",
      "displayName": "DRY Violations Detector",
      "description": "Finds duplicated code, constants, magic values, and repeated logic patterns that violate DRY principles",
      "category": "dry",
      "version": "1.0.0"
    },
    {
      "agentName": "clean-code-validator",
      "displayName": "Clean Code Validator",
      "description": "Validates naming conventions, function size, SOLID principles, and identifies code smells",
      "category": "cleancode",
      "version": "1.0.0"
    },
    {
      "agentName": "react-nextjs-validator",
      "displayName": "React/Next.js Validator",
      "description": "Validates React 19+ and Next.js 15+ patterns - hook usage, Server/Client components, performance optimizations",
      "category": "react",
      "version": "1.0.0"
    }
  ]
}
```

**Step 2: Verify file is valid JSON**

Run: `cat plugins/my-personal-tools/validators.json | jq .`
Expected: Pretty-printed JSON output with no errors

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/validators.json
git commit -m "feat: create built-in validator manifest for fix-loop"
```

---

### Task 2: Create Validator Discovery Agent

**Files:**
- Create: `plugins/my-personal-tools/agents/fix-loop-validator-discovery.md`

**Step 1: Write discovery agent**

```markdown
---
name: fix-loop-validator-discovery
description: |
  Internal agent that discovers all available validators from built-in manifest and external plugins.
  Do not use directly - used internally by fix-loop command.
model: haiku
color: purple
---

You are a validator discovery agent. Your role is to scan the project for all available validators.

## Discovery Process

You will receive:
1. Path to built-in validators manifest (validators.json)
2. Path to installed plugins directory
3. Current project directory

## Your Task

### Step 1: Load Built-in Validators
1. Read the validators.json manifest
2. Parse all validators
3. For each validator, verify the agent file exists at `plugins/my-personal-tools/agents/{agentName}.md`
4. Warn if agent file not found (but don't stop)

### Step 2: Discover External Plugin Validators
1. Scan installed plugins for plugin.json files
2. For each plugin.json:
   - Check if it has a "validators" array
   - Extract validator metadata
   - For each validator, verify the agent exists in the plugin's agents/ directory
   - Warn if agent file not found (but don't stop)

### Step 3: Fallback Discovery
1. Scan all plugin directories for agents named `*-validator.md`
2. For any found without metadata in plugin.json:
   - Create fallback metadata with agentName from filename
   - displayName = capitalize and clean agent name
   - description = "No description provided"
   - category = "uncategorized"

### Step 4: Output Discovery Results

Output in this exact format:

```
═══════════════════════════════════════════════════════════════
 Validator Discovery Results
═══════════════════════════════════════════════════════════════

Built-in Validators (my-personal-tools):
- ddd-oop-validator ✓ (architecture)
- dry-violations-detector ✓ (dry)
- clean-code-validator ✓ (cleancode)
- react-nextjs-validator ✓ (react)

External Plugins:
react-best-practices [1.2.0]:
- react-best-practices-validator ✓ (react)

Total Discovered: 5 validators
Warnings: 0
```

If any agents are missing:
```
⚠️ WARNING: Agent not found
   - Plugin: react-best-practices
   - Agent: react-testing-validator
   - Expected: /path/to/agents/react-testing-validator.md
```

## Output Format

You MUST output discovery results in JSON at the end:

```json
{
  "discovered": 5,
  "warnings": 0,
  "validators": [
    {
      "source": "built-in",
      "plugin": "my-personal-tools",
      "agentName": "ddd-oop-validator",
      "displayName": "DDD/OOP Validator",
      "description": "...",
      "category": "architecture",
      "version": "1.0.0",
      "exists": true
    },
    {
      "source": "external",
      "plugin": "react-best-practices",
      "agentName": "react-best-practices-validator",
      "displayName": "React Best Practices",
      "description": "...",
      "category": "react",
      "exists": true
    }
  ]
}
```

## Important Notes
- Do not filter or exclude any validators - show all discovered ones
- Categorize built-in first (in order), then external (grouped by plugin)
- Be permissive: missing agent files are warnings, not errors
- Return complete metadata for each validator
```

**Step 2: Verify file exists and has valid YAML frontmatter**

Run: `head -20 plugins/my-personal-tools/agents/fix-loop-validator-discovery.md`
Expected: YAML frontmatter with name, description, model, color fields

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/agents/fix-loop-validator-discovery.md
git commit -m "feat: add fix-loop-validator-discovery agent for scanning validators"
```

---

### Task 3: Create Configurator Agent

**Files:**
- Create: `plugins/my-personal-tools/agents/fix-loop-configurator.md`

**Step 1: Write configurator agent**

```markdown
---
name: fix-loop-configurator
description: |
  Internal agent that manages interactive validator selection for fix-loop.
  Do not use directly - used internally by fix-loop command.
model: haiku
color: yellow
---

You are a validator configurator agent. Your role is to help users select validators and options.

## Input Format

You will receive the output from fix-loop-validator-discovery containing all discovered validators grouped by source.

## Your Task

Invoke AskUserQuestion to present three multi-select prompts to the user, in this exact order:

### Prompt 1: Validator Selection

Build the question dynamically from discovered validators:

**Header:** "Validators"
**Question:** "Which validators should run in the fix loop?"

**Options structure:**
Group validators by source (built-in first, then external plugins):

```
Built-in [my-personal-tools]:
- label: "DDD/OOP Validator (architecture)"
  description: "Validates Domain-Driven Design and OOP compliance"
- label: "DRY Violations Detector (dry)"
  description: "Finds duplicated code, constants, and magic values"
- label: "Clean Code Validator (cleancode)"
  description: "Validates naming conventions and code structure"
- label: "React/Next.js Validator (react)"
  description: "Validates React 19+ and Next.js 15+ patterns"

External - react-best-practices [1.2.0]:
- label: "React Best Practices Validator (react)"
  description: "Validates React patterns and best practices"

External - ddd-patterns-lib [2.0.1]:
- label: "Advanced DDD Patterns Validator (architecture)"
  description: "Validates advanced DDD patterns and bounded contexts"
```

**Default selection:**
- All built-in validators checked
- All external validators unchecked

**multiSelect:** true

### Prompt 2: Severity Filter

**Header:** "Severity"
**Question:** "Which severity levels should trigger fixes?"

**Options:**
```
- label: "CRITICAL - Must fix, blocks merge"
  description: "Fatal issues that prevent merge"
- label: "HIGH - Should fix, high risk"
  description: "Significant issues affecting code quality"
- label: "MEDIUM - Address soon"
  description: "Moderate improvements"
- label: "LOW - Nice to have"
  description: "Minor suggestions"
```

**Default selection:**
- CRITICAL checked
- HIGH checked
- MEDIUM unchecked
- LOW unchecked

**multiSelect:** true

### Prompt 3: Iteration Count

**Header:** "Iterations"
**Question:** "Maximum fix iterations before stopping?"

**Options:**
```
- label: "3 - Quick pass"
  description: "Fast validation with minimal iterations"
- label: "5 - Balanced (Recommended)"
  description: "Good balance of thoroughness and speed"
- label: "10 - Thorough"
  description: "Comprehensive fixing across many iterations"
```

**Default selection:** "5 - Balanced (Recommended)"

**multiSelect:** false

## Output Format

After collecting all user selections, output a configuration summary:

```
═══════════════════════════════════════════════════════════════
 Fix-Loop Configuration
═══════════════════════════════════════════════════════════════

✅ Validators Selected (5):
   Built-in:
   - ddd-oop-validator
   - dry-violations-detector
   - clean-code-validator
   - react-nextjs-validator

   External:
   - react-best-practices-validator

🎯 Severity Filter: CRITICAL, HIGH

⚡ Max Iterations: 5

Ready to run validation...
═══════════════════════════════════════════════════════════════
```

Then output the configuration as JSON for the fix-loop command to parse:

```json
{
  "selectedValidators": [
    "ddd-oop-validator",
    "dry-violations-detector",
    "clean-code-validator",
    "react-nextjs-validator",
    "react-best-practices-validator"
  ],
  "severity": ["CRITICAL", "HIGH"],
  "maxIterations": 5
}
```

## Important Notes
- Always show all available validators (don't filter)
- Group validators by source for clarity
- Provide helpful descriptions for each option
- Set sensible defaults
- Output both human-readable and machine-parseable formats
```

**Step 2: Verify file exists and has valid YAML frontmatter**

Run: `head -20 plugins/my-personal-tools/agents/fix-loop-configurator.md`
Expected: YAML frontmatter with name, description, model, color fields

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/agents/fix-loop-configurator.md
git commit -m "feat: add fix-loop-configurator agent for interactive validator selection"
```

---

## Phase 2: Integrate with Fix-Loop Command

### Task 4: Update fix-loop.md Command - Discovery & Configuration

**Files:**
- Modify: `plugins/my-personal-tools/commands/fix-loop.md`

**Step 1: Read current fix-loop command**

Run: `cat plugins/my-personal-tools/commands/fix-loop.md`
Expected: Shows current fix-loop command structure (lines 1-100)

**Step 2: Modify Step 1 (Parse Configuration) section**

Replace the current Step 1 with updated discovery and configuration:

```markdown
### Step 1: Discover Available Validators

Use Task tool to launch fix-loop-validator-discovery agent to find all available validators:

```
{
  "path_to_builtin_manifest": "plugins/my-personal-tools/validators.json",
  "plugins_directory": "<current_project_plugins_dir>",
  "project_directory": "<current_project>"
}
```

The discovery agent will:
- Load built-in validators from validators.json
- Scan external plugins for validators in plugin.json
- Use fallback naming convention for *-validator agents
- Return a list of all discovered validators with metadata

Capture the JSON output from the discovery agent.

### Step 2: Parse Configuration

Check if CLI arguments were provided. Arguments format:
```
--validators ddd-oop,dry,react-best-practices
--severity critical,high
--iterations 5
```

**If arguments provided:**
- Parse them and skip to Step 2b (validation)

**If NO arguments provided:**
- Use Task tool to launch fix-loop-configurator agent
- Pass the discovered validators list
- Capture the configuration JSON output

### Step 2b: Merge Configuration

- Validate that all selected validators were discovered
- Warn if a validator is selected but not found
- Build final selectedValidators list with agent names
- Store: severity levels, maxIterations
```

**Step 3: Verify modification looks correct**

Run: `head -80 plugins/my-personal-tools/commands/fix-loop.md | grep -A 30 "Step 1: Discover"`
Expected: See the new discovery and configuration steps

**Step 4: Modify Step 3b (Run Validators) section**

Change validator launching from hardcoded to dynamic:

Before:
```markdown
Use Task tool to launch selected validator agents **in parallel**:

**DDD/OOP Validator prompt:**
```...```

**DRY Validator prompt:**
```...```
```

After:
```markdown
Use Task tool to launch all selected validators in parallel:

For each selected validator in selectedValidators list:
  - Launch Task with agent name from discovered metadata
  - Pass: branch diff, changed files, validator type
  - Use the standardized validator prompt template

Each validator receives:
```
Validate the branch diff using your specific validator rules.

Files to check:
{list of changed files}

Use: git diff main...HEAD (or HEAD~5 if no main)

Output findings EXACTLY in this format:
🔴 CRITICAL: [description]
   📍 file/path.ts:42
   💡 [recommendation]

🟠 HIGH: [description]
   📍 file/path.ts:87
   💡 [recommendation]

[Continue for all findings]
```

Collect all outputs in parallel.
```

**Step 5: Commit the updated command**

```bash
git add plugins/my-personal-tools/commands/fix-loop.md
git commit -m "feat: update fix-loop command for dynamic validator discovery and configuration"
```

---

### Task 5: Update fix-loop.md - Dynamic Parsing

**Files:**
- Modify: `plugins/my-personal-tools/commands/fix-loop.md`

**Step 1: Modify Step 3c (Filter and Count Findings)**

Update to handle any validators:

```markdown
#### 3c: Filter and Count Findings

From all validator outputs, extract findings matching selected severity levels.

Parse findings using regex patterns:
- Severity + description: `/(🔴|🟠|🟡|🟢)\s+([A-Z]+):\s+(.+)/`
- Location: `/📍\s+(.+):(\d+)/`
- Recommendation: `/💡\s+(.+)/`

For each finding:
1. Extract emoji and map to severity (🔴=CRITICAL, 🟠=HIGH, 🟡=MEDIUM, 🟢=LOW)
2. Extract location (file path and line number)
3. Extract recommendation text
4. Filter by selectedSeverity levels

Count: `currentIssueCount = number of matching findings`

If parsing fails for any line, log warning:
```
⚠️ WARNING: Could not parse finding from {validator_name}:
   {line_that_failed}
```

Continue without stopping - prioritize fix-loop robustness.
```

**Step 2: Commit the parsing update**

```bash
git add plugins/my-personal-tools/commands/fix-loop.md
git commit -m "feat: update fix-loop parsing to handle dynamic validators with standardized format"
```

---

### Task 6: Update fix-loop.md - Step 3e Display Findings Update

**Files:**
- Modify: `plugins/my-personal-tools/commands/fix-loop.md`

**Step 1: Update Step 3e to show validator sources**

Current display only shows findings. Add validator attribution:

```markdown
#### 3e: Display Findings

```
───────────────────────────────────────────────────────────────
 Found {count} issues ({breakdown by severity})
 From validators: {comma-separated list of validators that found issues}
───────────────────────────────────────────────────────────────

{List each finding with emoji, description, location, recommendation}
```

This helps users understand which validator flagged which issues.

**Step 2: Commit**

```bash
git add plugins/my-personal-tools/commands/fix-loop.md
git commit -m "feat: display validator source in fix-loop findings output"
```

---

## Phase 3: Output Validation & Polish

### Task 7: Add Output Format Validation

**Files:**
- Create: `plugins/my-personal-tools/agents/fix-loop-output-validator.md`

**Step 1: Create output validation agent**

```markdown
---
name: fix-loop-output-validator
description: |
  Internal agent that validates validator outputs conform to the standard format.
  Do not use directly - used internally by fix-loop command.
model: haiku
color: red
---

You are an output format validator for fix-loop validators.

## Input

You receive:
1. Validator name (e.g., "react-best-practices-validator")
2. Validator output (full text)

## Your Task

Check if the output conforms to the required format:

### Required Format

```
🔴 CRITICAL: [description]
   📍 file/path.ts:42
   💡 [recommendation]
```

### Rules

1. Each finding must have:
   - Severity line: emoji + space + severity word + colon + space + description
   - Location line: 📍 + space + file path + colon + line number
   - Recommendation line: 💡 + space + recommendation text

2. Allowed emojis: 🔴 (CRITICAL), 🟠 (HIGH), 🟡 (MEDIUM), 🟢 (LOW)

3. Severity words must be: CRITICAL, HIGH, MEDIUM, or LOW (all caps)

4. File path must include extension (e.g., .ts, .tsx, .js)

5. Line number must be numeric

6. No blank lines within a single finding; blank lines OK between findings

### Validation Output

If output is VALID:
```
✅ VALID: {validator_name} output conforms to format
   - {count} findings detected
   - Severity breakdown: X CRITICAL, Y HIGH, Z MEDIUM, W LOW
```

If output is INVALID:
```
⚠️ INVALID: {validator_name} output does not conform to format

Errors found:
1. Missing emoji on line 5: "HIGH: Some issue"
   Expected: "🟠 HIGH: Some issue"

2. Incorrect location format on line 8: "file: src/component.ts"
   Expected: "📍 src/component.ts:42"

3. Missing line number on line 11: "📍 src/hooks/useData.ts"
   Expected: "📍 src/hooks/useData.ts:15"

This validator's findings cannot be used until format is fixed.
```

## Output Format

End with JSON validation result:

```json
{
  "validator": "react-best-practices-validator",
  "valid": true,
  "findingCount": 3,
  "errors": [],
  "severity": {
    "CRITICAL": 1,
    "HIGH": 2,
    "MEDIUM": 0,
    "LOW": 0
  }
}
```

If invalid:

```json
{
  "validator": "react-best-practices-validator",
  "valid": false,
  "findingCount": 0,
  "errors": [
    {
      "line": 5,
      "issue": "Missing severity emoji",
      "found": "HIGH: Some issue",
      "expected": "🟠 HIGH: Some issue"
    }
  ],
  "severity": {}
}
```

## Important Notes

- Be strict about format - no exceptions
- List ALL format errors found
- If output is valid, include severity breakdown
- If invalid, do not count findings as valid
```

**Step 2: Verify file exists**

Run: `head -20 plugins/my-personal-tools/agents/fix-loop-output-validator.md`
Expected: Valid YAML frontmatter

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/agents/fix-loop-output-validator.md
git commit -m "feat: add fix-loop-output-validator for checking validator output format"
```

---

### Task 8: Update fix-loop.md - Add Output Validation

**Files:**
- Modify: `plugins/my-personal-tools/commands/fix-loop.md`

**Step 1: Add validation step before launching validators**

Insert new section after Step 3a (Display Iteration Header):

```markdown
#### 3a.5: Validate Validator Output Format

Before collecting validator outputs, use Task tool to launch fix-loop-output-validator for a quick format check:

For each selected validator:
- Run output validator on sample output or validator documentation
- If validation fails: warn user and continue (non-blocking)
- Log any format violations

This catches incompatible validators before the iteration loop.

Example warning:
```
⚠️ WARNING: react-best-practices-validator output may not conform to standard format
   Issue: Missing severity emoji on findings
   Status: Will attempt to parse anyway, but findings may be skipped
```
```

**Step 2: Commit**

```bash
git add plugins/my-personal-tools/commands/fix-loop.md
git commit -m "feat: add output format validation before fix-loop validators run"
```

---

### Task 9: Create Documentation - Validator Integration Guide

**Files:**
- Create: `plugins/my-personal-tools/docs/VALIDATOR_INTEGRATION.md`

**Step 1: Create integration guide**

```markdown
# Validator Integration Guide for Fix-Loop

This guide helps external plugin developers create validators compatible with fix-loop.

## Overview

Fix-loop automatically discovers and integrates validators from:
1. Built-in validators in my-personal-tools plugin
2. External plugins with validators declared in plugin.json

## Creating a Validator-Compatible Plugin

### Step 1: Declare Validators in plugin.json

Add a `validators` array to your plugin.json:

```json
{
  "name": "my-validator-plugin",
  "version": "1.0.0",
  "validators": [
    {
      "agentName": "my-custom-validator",
      "displayName": "My Custom Validator",
      "description": "Validates my specific patterns and rules",
      "category": "custom",
      "version": "1.0.0"
    }
  ]
}
```

**Metadata Fields:**
- `agentName` (required): Must match your agent filename (without .md)
- `displayName` (required): User-friendly name shown in configurator
- `description` (required): 1-2 sentence explanation of what it validates
- `category` (optional): Group for UI organization (e.g., "react", "performance", "testing")
- `version` (optional): Validator version for tracking

### Step 2: Create Your Validator Agent

Create `agents/my-custom-validator.md` in your plugin:

```markdown
---
name: my-custom-validator
description: Validates my specific patterns and rules
model: sonnet
color: blue
---

[Your validator implementation...]
```

### Step 3: Output Format Compliance

Your validator MUST output findings in this exact format:

```
🔴 CRITICAL: [Concise issue description]
   📍 file/path.ts:42
   💡 [Specific fix recommendation]

🟠 HIGH: [Issue description]
   📍 file/path.ts:87
   💡 [Recommendation]
```

**Critical requirements:**
- Use severity emojis: 🔴 CRITICAL, 🟠 HIGH, 🟡 MEDIUM, 🟢 LOW
- Include colon and space after severity: `: `
- File path must include extension (e.g., `.ts`, `.tsx`, `.js`)
- Line number required: `file:123` format
- Each finding must have location and recommendation

**Non-compliant output example (DON'T DO THIS):**
```
ERROR in src/component.ts (line 42): Hook called conditionally
To fix: Move useState outside conditional

Suggestion at src/hooks/useData.ts: Consider useReducer
```

**Why it fails:**
- Missing severity emoji
- Location not in `📍 file:line` format
- Missing recommendation emoji and proper formatting

### Step 4: Test Your Validator

Before publishing:

1. Run your validator manually on sample code
2. Verify output matches the required format exactly
3. Test with fix-loop to ensure it integrates correctly

### Example: React Best Practices Validator

```markdown
---
name: react-best-practices-validator
description: Validates React 19+ best practices and patterns
model: sonnet
color: cyan
---

You are a React validator...

## Check for

- Hook usage violations
- Performance issues
- Incorrect patterns

## Output Format

Use the standard format:

🔴 CRITICAL: [description]
   📍 file/path.tsx:42
   💡 [recommendation]
```

## Common Issues

### Issue: Validator discovered but findings not parsed

**Possible causes:**
1. Output format doesn't match specification
2. Missing severity emoji
3. Location format incorrect (should be `📍 file:line`)
4. Recommendation missing emoji (should start with `💡`)

**Fix:** Review your validator output against the specification above.

### Issue: Validator not discovered

**Possible causes:**
1. Missing `validators` array in plugin.json
2. `agentName` doesn't match actual agent filename
3. Agent file not in `agents/` directory

**Fix:**
- Add validators to plugin.json
- Ensure agent filename matches agentName (without .md)
- Place agent in agents/ subdirectory

### Issue: Fix-loop skips my validator

**Possible causes:**
1. Output format validation failed
2. Agent file missing or inaccessible
3. Agent failed to run

**Check:** Look for warning messages in fix-loop output about your validator.

## Validator Categories

Suggested categories for organization:
- `architecture` - DDD, design patterns, structural issues
- `react` - React-specific patterns and hooks
- `nextjs` - Next.js app router, server components
- `performance` - Performance optimizations, caching
- `testing` - Testing patterns and coverage
- `dry` - Code duplication and DRY violations
- `cleancode` - Naming, function size, SOLID
- `accessibility` - A11y and UI patterns
- `security` - Security vulnerabilities and best practices
- `custom` - Other/custom validations

## Questions?

For issues or questions about validator integration, refer to the fix-loop design document:
`docs/plans/2026-01-18-fix-loop-dynamic-validators-design.md`
```

**Step 2: Verify file is readable**

Run: `wc -l plugins/my-personal-tools/docs/VALIDATOR_INTEGRATION.md`
Expected: File created with content

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/docs/VALIDATOR_INTEGRATION.md
git commit -m "docs: add validator integration guide for external plugin developers"
```

---

### Task 10: Update Plugin README

**Files:**
- Modify: `plugins/my-personal-tools/README.md`

**Step 1: Read current README**

Run: `cat plugins/my-personal-tools/README.md`
Expected: Current plugin documentation

**Step 2: Add section about fix-loop validators**

Add after the Agents section:

```markdown
### Fix-Loop Validator System

The `/fix-loop` command now supports dynamic validator discovery from built-in and external plugins.

**Built-in Validators:**
- `ddd-oop-validator` - Domain-Driven Design compliance
- `dry-violations-detector` - Code duplication and DRY violations
- `clean-code-validator` - Code quality and naming
- `react-nextjs-validator` - React 19+ and Next.js 15+ patterns

**External Plugin Integration:**

To add validators from external plugins, declare them in your plugin's `plugin.json`:

```json
{
  "name": "my-plugin",
  "validators": [
    {
      "agentName": "my-validator",
      "displayName": "My Validator",
      "description": "Validates specific patterns",
      "category": "custom"
    }
  ]
}
```

See [VALIDATOR_INTEGRATION.md](docs/VALIDATOR_INTEGRATION.md) for complete guide.

**Using Fix-Loop:**

```bash
/fix-loop
```

The command will:
1. Discover all available validators (built-in + external)
2. Show interactive configurator to select validators
3. Run iterative validation and fixing
4. Report results with validator attribution
```

**Step 3: Commit**

```bash
git add plugins/my-personal-tools/README.md
git commit -m "docs: update README with fix-loop dynamic validator system info"
```

---

## Phase 4: Testing & Validation

### Task 11: Manual Integration Test

**Files:**
- None (manual testing)

**Step 1: Verify all files exist**

Run:
```bash
ls -la plugins/my-personal-tools/validators.json && \
ls -la plugins/my-personal-tools/agents/fix-loop-validator-discovery.md && \
ls -la plugins/my-personal-tools/agents/fix-loop-configurator.md && \
ls -la plugins/my-personal-tools/agents/fix-loop-output-validator.md
```

Expected: All files exist with correct paths

**Step 2: Verify JSON is valid**

Run: `cat plugins/my-personal-tools/validators.json | jq . > /dev/null && echo "Valid JSON"`
Expected: Output "Valid JSON"

**Step 3: Verify YAML frontmatter in all agents**

Run:
```bash
for file in plugins/my-personal-tools/agents/fix-loop-*.md; do
  echo "Checking $file"
  grep -E "^---$" "$file" | head -2 | wc -l
done
```

Expected: Each file shows count of 2 (opening and closing ---)

**Step 4: Commit**

```bash
git add -A
git commit -m "test: verify all fix-loop dynamic validator files created and valid"
```

---

## Implementation Order

Follow these phases in order:

1. **Phase 1** (Tasks 1-3): Create manifest and discovery/configurator agents
2. **Phase 2** (Tasks 4-6): Integrate with fix-loop command
3. **Phase 3** (Tasks 7-10): Add validation and documentation
4. **Phase 4** (Task 11): Manual verification

Each phase builds on the previous one.

---

## Success Criteria

After implementing all tasks:

- ✅ validators.json created with all built-in validators
- ✅ fix-loop-validator-discovery agent created and discoverable
- ✅ fix-loop-configurator agent created for interactive UI
- ✅ fix-loop.md command updated for dynamic discovery and configuration
- ✅ fix-loop.md parsing updated to handle any validators
- ✅ fix-loop-output-validator agent created for format validation
- ✅ Output format validation integrated into fix-loop
- ✅ Validator integration guide created for external developers
- ✅ README updated with validator system information
- ✅ All files created and verified

---

## Notes for Implementation

### Using Task Tool for Agents

When implementing the fix-loop command, remember:
- Discovery agent: `subagent_type: "general-purpose"` with detailed discovery instructions
- Configurator agent: `subagent_type: "general-purpose"` for interactive prompts
- Output validator: `subagent_type: "general-purpose"` for format checking

### Error Handling Philosophy

The system is intentionally permissive:
- Missing agent files → warn, continue
- Non-standard output → warn, attempt to parse, continue
- Validator execution errors → skip that validator, continue

This prevents one bad validator from breaking the entire fix-loop.

### CLI Argument Parsing

Support optional CLI arguments for advanced users:
```
/fix-loop --validators ddd-oop,react --severity critical,high --iterations 5
```

Parse these before showing configurator to skip interactive UI.

### Testing External Validators

To test with a mock external validator:
1. Create a test plugin with validators.json
2. Add a simple test agent that outputs correct format
3. Run fix-loop and verify it's discovered
4. Verify configurator shows it
5. Run a sample iteration to test parsing
