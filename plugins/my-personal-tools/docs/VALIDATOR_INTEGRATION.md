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
