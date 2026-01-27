---
name: manage-skills
description: Create and improve Claude Skills with proper structure, frontmatter, and best practices. Use when creating new skills, improving existing skills, refactoring skill content, or scaffolding skill files.
---

# Managing Claude Skills

## Creating Skills

### Quick Start

Create a skill by making a `SKILL.md` file in `.claude/skills/<skill-name>/`:

```markdown
---
name: my-skill-name
description: Brief description of what this skill does and when to use it.
---

# My Skill Name

## Instructions
[Your instructions here]
```

## Required Structure

Every skill needs:

1. **SKILL.md file** with YAML frontmatter
2. **name field**: lowercase letters, numbers, hyphens only (max 64 chars)
3. **description field**: what the skill does AND when to use it (max 1024 chars)

## Skill Directory Layout

```
.claude/skills/
└── my-skill/
    ├── SKILL.md          # Main instructions (required)
    ├── reference.md      # Additional details (optional)
    └── scripts/          # Utility scripts (optional)
        └── helper.py
```

## Writing the Description

The description is critical for skill discovery. Include:
- What the skill does
- When Claude should use it
- Key trigger words

**Good example:**
```yaml
description: Generate git commit messages by analyzing staged changes. Use when committing code, writing commit messages, or reviewing diffs.
```

**Bad example:**
```yaml
description: Helps with git stuff
```

## Skill Templates

### Template 1: Instructions-Only Skill

For guidance-based skills without code:

```markdown
---
name: code-review
description: Review code for bugs, style, and best practices. Use when reviewing pull requests, analyzing code quality, or checking for issues.
---

# Code Review

## Process

1. Read the code thoroughly
2. Check for bugs and edge cases
3. Verify style consistency
4. Suggest improvements

## Checklist

- [ ] No obvious bugs
- [ ] Error handling present
- [ ] Consistent naming
- [ ] No security issues
```

### Template 2: Skill with Scripts

For skills that include executable utilities:

```markdown
---
name: data-validation
description: Validate data files against schemas. Use when checking CSV, JSON, or config file formats.
---

# Data Validation

## Usage

Run validation:
```bash
python scripts/validate.py input.json schema.json
```

## Scripts

- **validate.py**: Check data against schema
- **format.py**: Auto-fix formatting issues

See [reference.md](reference.md) for schema format details.
```

### Template 3: Domain-Specific Skill

For skills with extensive reference material:

```markdown
---
name: api-integration
description: Integrate with the Acme API. Use when calling Acme endpoints, handling Acme webhooks, or processing Acme data.
---

# Acme API Integration

## Quick Reference

Base URL: `https://api.acme.com/v2`
Auth: Bearer token in header

## Common Operations

**Get user**: `GET /users/{id}`
**Create order**: `POST /orders`

For complete endpoint docs, see [endpoints.md](endpoints.md).
For authentication details, see [auth.md](auth.md).
```

## Best Practices

### Be Concise
Claude is smart. Only include information Claude doesn't already know.

### Use Progressive Disclosure
- Keep SKILL.md under 500 lines
- Put detailed reference material in separate files
- Link with relative paths: `[details](reference.md)`

### Set Appropriate Freedom

**High freedom** (guidelines):
```markdown
Review the code for potential issues and suggest improvements.
```

**Low freedom** (specific steps):
```markdown
Run exactly: `python scripts/migrate.py --verify --backup`
Do not modify the command.
```

### Include Workflows for Complex Tasks

```markdown
## Deployment Workflow

1. Run tests: `bun test`
2. Build: `bun run build`
3. If build fails, fix errors and repeat from step 1
4. Deploy: `bun run deploy`
```

## Common Mistakes to Avoid

- Using vague descriptions
- Including time-sensitive information
- Using Windows-style paths (use forward slashes)
- Offering too many options without a default
- Deeply nesting file references (keep one level deep)

## File Naming

Use descriptive, lowercase names with hyphens:
- `api-reference.md` (good)
- `APIREF.md` (bad)
- `scripts/validate-schema.py` (good)
- `scripts/vs.py` (bad)

## Testing Your Skill

1. Create the skill files
2. Start a new conversation
3. Make a request that should trigger the skill
4. Verify Claude uses the skill correctly
5. Iterate based on behavior

---

## Improving Existing Skills

### When to Improve a Skill

- Skill produces inconsistent results
- Instructions are ambiguous or unclear
- Missing edge cases or workflows
- Description doesn't trigger skill when expected
- Skill has grown too large and needs refactoring

### Improvement Process

1. **Read the existing skill** - Understand current structure and intent
2. **Identify the problem** - What's not working or missing?
3. **Make targeted changes** - Don't rewrite unless necessary
4. **Test the changes** - Verify improvements in a new conversation

### Common Improvements

#### Sharpen the Description

If the skill isn't triggering when expected, improve the description:

```yaml
# Before - too vague
description: Helps with database tasks

# After - specific triggers
description: Generate and run database migrations. Use when creating tables, modifying schemas, or updating database structure.
```

#### Add Missing Workflows

If Claude doesn't handle certain cases, add explicit workflows:

```markdown
## Edge Cases

### Empty Input
If input is empty, prompt user for required data before proceeding.

### Existing File Conflict
If file already exists, ask user whether to overwrite, merge, or abort.
```

#### Reduce Ambiguity

Replace vague instructions with specific ones:

```markdown
# Before - ambiguous
Review the code and suggest improvements.

# After - specific
Review the code for:
1. Security vulnerabilities (injection, XSS, auth issues)
2. Performance problems (N+1 queries, unnecessary loops)
3. Error handling gaps

For each issue found, provide:
- Location (file:line)
- Problem description
- Suggested fix
```

#### Split Large Skills

If a skill exceeds 500 lines, consider:

1. Moving reference material to separate files
2. Splitting into multiple focused skills
3. Using a reference folder structure:

```
my-skill/
├── SKILL.md           # Core instructions only
└── reference/
    ├── api.md         # API details
    ├── examples.md    # Extended examples
    └── errors.md      # Error handling guide
```

#### Improve Examples

Add before/after examples to clarify expected behavior:

```markdown
## Examples

**Input**: User asks "format this JSON"
**Expected**: Format JSON with 2-space indentation, sorted keys

**Input**: User provides malformed JSON
**Expected**: Report specific syntax error location, suggest fix
```

### Refactoring Checklist

- [ ] Description clearly states what and when
- [ ] Instructions are unambiguous
- [ ] Edge cases are documented
- [ ] Examples demonstrate expected behavior
- [ ] Reference material is in separate files
- [ ] File is under 500 lines

## Additional Resources

- [Frontmatter Rules](frontmatter-rules.md) - Detailed YAML validation rules
- [Advanced Patterns](advanced-patterns.md) - Feedback loops, workflows, scripts

