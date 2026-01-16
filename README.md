# claude-skills

Personal Claude Code marketplace containing custom skills, agents, commands, and hooks.

## Installation

Add this marketplace to Claude Code:

```bash
/plugin marketplace add markky21/claude-skills
```

Then install the plugin:

```bash
/plugin install my-personal-tools@claude-skills
```

## Structure

```
claude-skills/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    └── my-personal-tools/
        ├── skills/      # Workflow instructions
        ├── agents/      # Specialized agents
        ├── commands/    # Slash commands
        └── hooks/       # Event handlers
```

## Adding New Content

### Skills
Create a new directory in `plugins/my-personal-tools/skills/` with a `SKILL.md` file.

### Commands
Create a new `.md` file in `plugins/my-personal-tools/commands/`.

### Agents
Create a new `.md` file in `plugins/my-personal-tools/agents/`.

### Hooks
Add hook configuration to `plugins/my-personal-tools/.claude-plugin/plugin.json`.
