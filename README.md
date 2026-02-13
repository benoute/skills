# Skills

A collection of [Agent Skills](https://agentskills.io/specification) for AI coding agents.

## Available Skills

| Skill | Description |
|-------|-------------|
| [go-nerd-review](skills/go-nerd-review/) | Review and audit existing Go code for performance issues, missed idioms, and unnecessary complexity |
| [go-nerd-write](skills/go-nerd-write/) | Write performant, minimal Go code with data-oriented design and modern Go 1.26 idioms |

## Installation

Copy the skills you need into your agent's skills directory (e.g. `~/.agents/skills/`), or point your agent configuration to this repository.

## Structure

Each skill follows the Agent Skills specification:

```
skills/<skill-name>/
  SKILL.md              # Skill definition (frontmatter + instructions)
  references/           # Supporting reference documents
```
