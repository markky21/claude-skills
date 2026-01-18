---
description: Manage external skills - add, list, or remove skills from other repositories
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# External Skill Management Command

Manage external skills that are bundled with this plugin. External skills come from other repositories and can be conditionally activated based on project context.

## Usage

```
/external-skill add <source> [--condition <type>]
/external-skill list
/external-skill remove <source>
```

## Workflow

### 1. Determine Action

Ask the user what they want to do if not clear from context:
- **add**: Add a new external skill source
- **list**: Show all configured external skills
- **remove**: Remove an external skill source

### 2. For Adding External Skills

When adding a new external skill:

1. **Get the source**: GitHub repo format `owner/repo` (e.g., `vercel-labs/agent-skills`)

2. **Ask about conditions**: External skills can be conditionally activated. Ask:
   - Should this skill always be active, or only in certain projects?
   - Common conditions:
     - **React/Next.js projects**: Activate when `next.config.*` exists or `react`/`next` in dependencies
     - **TypeScript projects**: Activate when `tsconfig.json` exists
     - **Node.js projects**: Activate when `package.json` exists
     - **Python projects**: Activate when `requirements.txt` or `pyproject.toml` exists
     - **Always active**: No conditions, always available

3. **Update plugin.json**: Add the external skill to the `externalSkills` array

### 3. Condition Schema

```json
{
  "source": "owner/repo",
  "description": "Brief description of what this skill provides",
  "skills": "*",
  "condition": {
    "type": "any",
    "rules": [
      { "files": ["pattern1", "pattern2"] },
      { "dependencies": ["dep1", "dep2"] },
      { "devDependencies": ["devDep1"] }
    ]
  }
}
```

**Condition Types:**
- `"any"`: Skill activates if ANY rule matches (OR logic)
- `"all"`: Skill activates only if ALL rules match (AND logic)

**Rule Types:**
- `files`: Glob patterns to match against project files
- `dependencies`: Check package.json dependencies
- `devDependencies`: Check package.json devDependencies

### 4. Common Condition Templates

**React/Next.js:**
```json
{
  "type": "any",
  "rules": [
    { "files": ["next.config.*", "next.config.ts", "next.config.js"] },
    { "dependencies": ["next", "react"] },
    { "devDependencies": ["next", "react"] }
  ]
}
```

**TypeScript:**
```json
{
  "type": "any",
  "rules": [
    { "files": ["tsconfig.json", "tsconfig.*.json"] }
  ]
}
```

**Python:**
```json
{
  "type": "any",
  "rules": [
    { "files": ["requirements.txt", "pyproject.toml", "setup.py", "Pipfile"] }
  ]
}
```

**Always Active (no condition):**
```json
{
  "source": "owner/repo",
  "description": "Description",
  "skills": "*"
}
```

### 5. For Listing External Skills

Read the plugin.json and display:
- Source repository
- Description
- Activation conditions (if any)
- Specific skills included (or "*" for all)

### 6. For Removing External Skills

1. List current external skills
2. Confirm which one to remove
3. Update plugin.json to remove the entry

## Example Interactions

**Adding a React skill:**
```
User: /external-skill add vercel-labs/agent-skills
Assistant: I'll add vercel-labs/agent-skills. Should this be:
1. Always active
2. Only for React/Next.js projects (recommended for this skill)
3. Custom condition

User: Only for React/Next.js
Assistant: Added vercel-labs/agent-skills with React/Next.js condition.
```

**Listing skills:**
```
User: /external-skill list
Assistant: External skills configured:
1. vercel-labs/agent-skills
   - Condition: React/Next.js projects (next.config.* or react/next dependencies)
   - Skills: All (*)
```

## Plugin.json Location

The plugin configuration is at:
`/home/user/claude-skills/plugins/my-personal-tools/.claude-plugin/plugin.json`

Always read this file first before making changes.
