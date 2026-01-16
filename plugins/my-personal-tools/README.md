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

## Directories

- **skills/**: Workflow instructions and best practices
- **commands/**: Slash commands
- **agents/**: Specialized agent definitions
- **hooks/**: Event-triggered automations (empty)
