# Firstloop Claude Code Plugins

A collection of Claude Code plugins maintained by Firstloop.

## Installation

Add this repository to your Claude Code plugin sources.

## Plugins

### Core

Core Claude Code plugins for Firstloop projects.

**Skills:**

- **create-skill** - Create new Claude Skills with proper structure, frontmatter, and best practices. Use when you want to create, build, or scaffold a new skill. Includes templates for instruction-only skills, skills with scripts, and domain-specific skills.

## Repository Structure

```
claude-code-plugins/
├── .claude-plugin/
│   └── marketplace.json      # Plugin marketplace configuration
└── plugins/
    └── core/                 # Core plugin
        ├── .claude-plugin/
        │   └── plugin.json   # Plugin metadata
        └── skills/
            └── create-skill/ # Skill files
```

## License

MIT
