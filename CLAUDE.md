# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A library of reusable skill definitions for Claude Code and Windsurf. Each skill encodes hard-won lessons for a specific task (CI/CD setup, auth, security, deployments) so future agent runs avoid known pitfalls.

## Directory Layout

| Directory | Contents |
|---|---|
| `saved_claude_skills/<name>/SKILL.md` | Claude Code skills invoked via slash commands |
| `saved_windsurf_skills/<name>/SKILL.md` | Windsurf skills |
| `saved_windsurf_workflows/<name>/WORKFLOW.md` | Windsurf workflows |
| `saved_windsurf_rules/<name>/RULE.md` | Windsurf always-on rules |
| `saved_windsurf_plans/<name>/PLAN.md` | Windsurf app development plans |

## Skill File Format

Every `SKILL.md` starts with YAML frontmatter:

```markdown
---
name: skill-name
description: One-line description shown in skill listings
trigger: /skill-name   # omit for context-triggered skills
---
```

Followed by:
1. Background — hard-won lessons with specific failure modes and fixes
2. Step-by-step action instructions Claude should follow

## How Claude Code Skills Are Installed

Claude Code skills live in `~/.claude/skills/<name>/SKILL.md`. After adding a new skill here, the user must also:
1. Copy or symlink the `SKILL.md` to `~/.claude/skills/<name>/SKILL.md`
2. Register a trigger in `~/.claude/CLAUDE.md` so the skill appears in the skills list

The `README.md` is the human-readable catalog of all skills — keep it in sync when adding or modifying skills.

## Adding a New Skill

1. Create `saved_claude_skills/<name>/SKILL.md` with the frontmatter and content
2. Add a summary entry to `README.md` under the appropriate section
3. The README entry should include: trigger, what it does, and key lessons (not just a description)
