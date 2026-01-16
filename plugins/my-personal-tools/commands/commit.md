---
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)
description: Create a git commit
---

## Context

- Current git status: !`git status`
- Current git diff (staged and unstaged changes): !`git diff HEAD`
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -10`

## Your task

Based on the above changes, create a single git commit with a conventional commits header only (no body, no footer).

**Commit message format:**
- Use conventional commits: `type(scope): description`
- Types: feat, fix, refactor, docs, test, chore, etc.
- Only the header line - no body text, no blank lines, no Co-Authored-By footer
- Example: `feat(auth): add login form validation`

You have the capability to call multiple tools in a single response. Stage and create the commit using a single message. Do not use any other tools or do anything else. Do not send any other text or messages besides these tool calls.
