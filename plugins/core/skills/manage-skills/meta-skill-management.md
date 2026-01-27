# Meta Skill Management

This document describes how to contribute improvements to shared skills in the claude-code-plugins repository.

## Repository

**GitHub**: https://github.com/firstloophq/claude-code-plugins

All shared skills (core and monotemplate plugins) are managed in this repository.

## Creating Issues for Skill Improvements

When you identify a skill that needs improvement, create a GitHub issue to track the work.

### When to Create an Issue

- Skill behavior is inconsistent or incorrect
- Missing functionality or workflows
- Description needs improvement for better triggering
- Documentation is unclear or outdated
- New skill idea for the shared plugins

### Issue Format

```bash
gh issue create --repo firstloophq/claude-code-plugins --title "Brief description" --body "$(cat <<'EOF'
## Skill
[Plugin name]:[Skill name] (e.g., `core:manage-skills` or `monotemplate:crud`)

## Problem
[What's wrong or missing]

## Proposed Solution
[How to fix or improve it]

## Examples
[Optional: Before/after examples or specific use cases]
EOF
)"
```

### Triggering Claude to Work on Issues

Tag `@claude` in your issue or comment to trigger a GitHub Action that runs Claude on the issue automatically.

**In the issue body:**
```markdown
@claude Please update the pm skill to add a new workflow for sprint planning.
```

**In a comment on an existing issue:**
```markdown
@claude Can you improve the description for the crud skill so it triggers when users ask about "adding a new model"?
```

Claude will read the issue context and respond with analysis, proposed changes, or a PR.

### Labels

Use appropriate labels when creating issues:
- `skill-improvement` - Improving an existing skill
- `new-skill` - Proposing a new skill
- `bug` - Skill is broken or behaving incorrectly
- `documentation` - Docs-only changes

## Quick Commands

**List open skill issues:**
```bash
gh issue list --repo firstloophq/claude-code-plugins
```

**View a specific issue:**
```bash
gh issue view <issue-number> --repo firstloophq/claude-code-plugins
```

**Create a PR for skill changes:**
```bash
gh pr create --repo firstloophq/claude-code-plugins --title "Improve [skill-name]" --body "Fixes #<issue-number>"
```

## Plugin Structure

```
claude-code-plugins/
├── plugins/
│   ├── core/                    # Core skills for all projects
│   │   └── skills/
│   │       ├── manage-skills/   # This skill
│   │       └── pm/              # Product manager skill
│   └── monotemplate/            # Skills for monorepo template
│       └── skills/
│           └── crud/            # CRUD architecture skill
```

## Contribution Workflow

1. **Identify improvement** - Notice a skill issue while working
2. **Create issue** - Document the problem in the repo
3. **Branch and fix** - Create a branch, make changes
4. **Test locally** - Verify the skill works as expected
5. **Create PR** - Submit for review
6. **Merge** - Once approved, merge to main
