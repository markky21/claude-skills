# Fix Loop Command Design

## Overview

An iterative validation and fix command inspired by the [Ralph Wiggum technique](https://awesomeclaude.ai/ralph-wiggum). Runs validators in a loop, automatically fixing issues until no critical/high severity findings remain.

## Command

**Name:** `/fix-loop`

**Purpose:** Iteratively validate and fix code issues until no critical/high severity findings remain, or max iterations reached.

## Flow

```
┌─────────────────────────────────────────────────────────────┐
│  1. Parse CLI args OR show interactive prompts              │
│  2. Get branch diff (changed files vs main)                 │
│  3. LOOP:                                                   │
│     a. Run selected validators in parallel                  │
│     b. Filter findings by selected severity                 │
│     c. If no findings → SUCCESS, exit                       │
│     d. If max iterations reached → STOP, show remaining     │
│     e. Display what will be fixed                           │
│     f. Apply fixes                                          │
│     g. Increment iteration counter                          │
│     h. Go to step 3a                                        │
│  4. Show summary (iterations used, issues fixed, remaining) │
└─────────────────────────────────────────────────────────────┘
```

## Configuration

### Interactive Prompts (default)

When no CLI args provided, show:

1. **Validators** (multi-select, defaults checked):
   - ☑️ DDD/OOP Validator
   - ☑️ DRY Violations
   - ☐ Clean Code
   - ☐ Architecture Compliance

2. **Severity threshold** (multi-select, defaults checked):
   - ☑️ 🔴 CRITICAL
   - ☑️ 🟠 HIGH
   - ☐ 🟡 MEDIUM
   - ☐ 🟢 LOW

3. **Max iterations**: Default `5`

### CLI Arguments

```bash
# Skip prompts with args
/fix-loop --validators ddd-oop,dry,clean-code --severity critical,high --iterations 10

# Use defaults (still shows prompts)
/fix-loop
```

### Validator Mappings

| CLI name | Agent |
|----------|-------|
| `ddd-oop` | ddd-oop-validator |
| `dry` | dry-violations-detector |
| `clean-code` | clean-code-validator |
| `architecture` | architecture-validator (inline) |

## Output Format

### Each Iteration

```
═══════════════════════════════════════════════════════════════
 🔄 Iteration 1/5
═══════════════════════════════════════════════════════════════

📋 Running validators: DDD/OOP, DRY
   Scope: 8 files changed vs main

⏳ Validating...

───────────────────────────────────────────────────────────────
 Found 3 issues (2 CRITICAL, 1 HIGH)
───────────────────────────────────────────────────────────────

🔴 CRITICAL: Anemic domain model
   📍 src/domain/order.ts:15
   💡 Order class has only data, no behavior methods

🔴 CRITICAL: Tell/Don't Ask violation
   📍 src/services/order.service.ts:42
   💡 Extracting order.items to calculate total externally

🟠 HIGH: Parameter bloat
   📍 src/services/pricing.service.ts:28
   💡 6 primitive parameters, should pass domain objects

───────────────────────────────────────────────────────────────
 🔧 Fixing 3 issues...
───────────────────────────────────────────────────────────────

✅ Fixed: Added getTotal(), addItem(), confirm() to Order entity
✅ Fixed: Moved calculation from OrderService to Order.getTotal()
✅ Fixed: Replaced primitives with PricingContext object

→ Continuing to iteration 2...
```

### Final Summary

```
═══════════════════════════════════════════════════════════════
 ✅ Fix Loop Complete
═══════════════════════════════════════════════════════════════

Iterations: 3/5
Issues fixed: 7
Remaining: 0

🎉 No CRITICAL or HIGH issues remaining!
```

## Implementation

### File Structure

```
plugins/my-personal-tools/
├── commands/
│   └── fix-loop.md           # New command
├── agents/
│   ├── ddd-oop-validator.md      # Existing
│   ├── dry-violations-detector.md # Existing
│   ├── clean-code-validator.md    # Existing
│   └── fix-loop-fixer.md          # New - handles fixing phase
```

### Phases

1. **Validation phase** - Launch selected validator agents in parallel using Task tool
2. **Parsing phase** - Extract severity + location from agent outputs (regex for 🔴/🟠/🟡/🟢 patterns)
3. **Fix phase** - Launch fixer agent that receives findings and applies fixes
4. **Loop control** - Track iteration count, check termination conditions

### Fixer Agent

- Receives list of findings with file:line and recommended fixes
- Reads each file, applies the fix
- Reports what was changed

### Termination Conditions

- No findings matching selected severity → SUCCESS
- Max iterations reached → STOP with remaining issues listed
- Fixer fails to reduce issue count for 2 consecutive iterations → STOP (prevents infinite loops)

## Safety

- Default max iterations: 5
- Stall detection: stops if no progress for 2 iterations
- Branch diff scope: only touches files changed vs main
- Transparent: shows what will be fixed before fixing
