# my-personal-tools

Personal productivity tools for Claude Code: DDD/OOP patterns, React/Next.js validation, DRY detection, code review, and git workflows.

## Commands

| Command | Description |
|---------|-------------|
| `/commit` | Create conventional commit with staged changes |
| `/review-branch` | Run comprehensive multi-agent code review on current branch |

## Skills

| Skill | Description |
|-------|-------------|
| `ddd-oop` | Domain-Driven Design & OOP patterns - rich domain models, aggregates, value objects |
| `react-nextjs-validator` | React 19+ & Next.js 15+ best practices, hooks selection, performance optimization |
| `dry-violations` | DRY violation detection - search before implementing, avoid duplicate constants/enums/logic |
| `review-branch` | Comprehensive branch review process - architecture, DDD/OOP, DRY, clean code, tests |

## Agents

Specialized subagents that can be launched via Task tool for focused validation:

| Agent | Description | Color |
|-------|-------------|-------|
| `ddd-oop-validator` | Validates DDD/OOP compliance - detects anemic models, Tell/Don't Ask violations | Blue |
| `dry-violations-detector` | Finds code duplication, magic values, repeated patterns | Orange |
| `react-nextjs-validator` | Validates React 19/Next.js 15 patterns, hook usage, Server/Client split | Cyan |
| `clean-code-validator` | Checks naming, function size, SOLID principles, code smells | Green |

### Using Agents

Agents are automatically available when you install this plugin. Use them via:

```
Use the Task tool to launch the ddd-oop-validator agent to check my recent changes.
```

Or reference them in your prompts:

```
Please use the dry-violations-detector agent to scan for duplications in this branch.
```

The `/review-branch` command orchestrates all agents in parallel for comprehensive review.

## Fix-Loop Validator System

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
      "agentName": "my-validator",           // Required: must match agent filename
      "displayName": "My Validator",         // Required: shown in configurator
      "description": "Validates specific patterns",  // Required: 1-2 sentence description
      "category": "custom"                   // Optional: for UI grouping
    }
  ]
}
```

**Field Documentation:**
- **agentName** (Required): Must match the agent filename (without .md extension)
- **displayName** (Required): Human-readable name shown in the configurator UI
- **description** (Required): Brief 1-2 sentence explanation of what the validator checks
- **category** (Optional): Grouping category for UI organization
- **version** (Optional): For tracking validator updates

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

**Validator Output Format:**

**Important:** Spacing is critical for parsing. External validators must conform to exact format requirements.

All validators must output findings in this standard format:

```
🔴 CRITICAL: [issue description]
   📍 file/path.ts:42
   💡 [fix recommendation]
```

**Critical Requirements:**
- Severity emoji: 🔴 (CRITICAL), 🟠 (HIGH), 🟡 (MEDIUM), 🟢 (LOW)
- Exactly ONE space after emoji before severity word
- Location line: exactly THREE spaces before 📍, ONE space after
- Recommendation line: exactly THREE spaces before 💡, ONE space after
- Missing or incorrect spacing causes parsing failures

External validators that don't conform to this format will be flagged during discovery.

**Before Publishing Your Validator:**
See [VALIDATOR_INTEGRATION.md](docs/VALIDATOR_INTEGRATION.md) Step 5 for how to validate your output format using the fix-loop-output-validator agent.

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

## Directories

- **skills/**: Workflow instructions and best practices
- **commands/**: Slash commands
- **agents/**: Specialized agent definitions
- **hooks/**: Event-triggered automations (empty)
