---
name: pm
description: Product Manager skill for planning, implementing, and documenting features. Use when creating GitHub issues from specs, implementing planned work, or maintaining project documentation.
---

# Product Manager

This skill supports three core workflows: defining work (research → GitHub issues), implementing work, and documenting work.

## 1. Define (Research → GitHub Issue)

Use this workflow when the user wants to plan a feature or task.

### Process

1. **Accept the prompt** - Understand what the user wants to build or change
2. **Research** - Explore the codebase to understand:
   - Existing patterns and architecture
   - Files that would be affected
   - Dependencies and constraints
3. **Ask clarifying questions** - Iterate with the user on:
   - Scope and requirements
   - Edge cases
   - Acceptance criteria
4. **Draft the plan** - Present a summary for user approval
5. **Create GitHub issue** - Once approved, create the issue using `gh`

### GitHub Issue Format

```bash
gh issue create --title "Brief descriptive title" --body "$(cat <<'EOF'
## Summary
[1-2 sentence overview]

## Requirements
- [ ] Requirement 1
- [ ] Requirement 2

## Technical Approach
[Brief description of implementation approach]

## Files Affected
- `path/to/file1.ts`
- `path/to/file2.ts`

## Acceptance Criteria
- [ ] Criteria 1
- [ ] Criteria 2

## Notes
[Any additional context, constraints, or considerations]
EOF
)"
```

### Key Principles

- Keep asking questions until requirements are clear
- Research the codebase before proposing solutions
- Get explicit user approval before creating the issue
- Be specific about files and changes in the technical approach

---

## 2. Implement

Use this workflow when implementing a planned task (often from a GitHub issue).

### Process

1. Read and understand the plan/issue
2. Implement the changes following existing codebase patterns
3. Test the implementation
4. Update documentation if needed (see Documentation workflow)

*Note: Implementation details depend on the specific task and codebase patterns.*

---

## 3. Document

Use this workflow when creating or updating project documentation.

### Documentation Structure

All documentation lives in `/docs` at the repository root:

```
docs/
├── index.md           # Required: Index of all documentation
├── architecture.md    # Example: System architecture
├── api/               # Example: API documentation folder
│   ├── endpoints.md
│   └── authentication.md
└── guides/            # Example: User/dev guides folder
    └── getting-started.md
```

### Index File (docs/index.md)

Always maintain an index that lists all documentation:

```markdown
# Documentation Index

## Overview
- [Architecture](architecture.md) - System architecture overview

## API
- [Endpoints](api/endpoints.md) - API endpoint reference
- [Authentication](api/authentication.md) - Auth flows and tokens

## Guides
- [Getting Started](guides/getting-started.md) - Setup and first steps
```

### Documentation Process

1. **Check if docs/ exists** - Create it if needed
2. **Read docs/index.md** - Understand existing documentation
3. **Create/update the relevant doc** - Use clear markdown formatting
4. **Update docs/index.md** - Add entry for new docs, update descriptions for changed docs

### Documentation Guidelines

- Use clear, descriptive file names (e.g., `api-authentication.md` not `auth.md`)
- Keep related docs in folders (e.g., `api/`, `guides/`)
- Always update the index when adding or removing docs
- Use relative links between documentation files
