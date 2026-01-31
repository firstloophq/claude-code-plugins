---
name: gh-action-config
description: Configure the anthropics/claude-code-action GitHub Action correctly. Use when setting up Claude Code workflows, troubleshooting GitHub Action configuration, or passing CLI flags via claude_args.
---

# Claude Code GitHub Action Configuration Guide

This skill helps you properly configure the `anthropics/claude-code-action` GitHub Action, avoiding common pitfalls with inputs and CLI flags.

## Key Concept: Action Inputs vs CLI Flags

The most common mistake is confusing **action inputs** with **CLI flags**. They are NOT interchangeable.

- **Action inputs**: Defined in workflow YAML directly under `with:`
- **CLI flags**: Must be passed through the `claude_args` input

## Common Mistake: Invalid Direct Inputs

These parameters look like they should work as action inputs, but they will be **ignored with warnings**:

```yaml
# ❌ WRONG - These are NOT valid action inputs
- uses: anthropics/claude-code-action@v1
  with:
    model: claude-opus-4-5-20251101           # ❌ Ignored
    allowed_tools: "Bash(bun *)"              # ❌ Ignored
    append_system_prompt: "Your prompt"       # ❌ Ignored
```

## Correct Approach: Use claude_args

Pass CLI flags through the `claude_args` input:

```yaml
# ✅ CORRECT - Pass CLI flags via claude_args
- uses: anthropics/claude-code-action@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

    claude_args: >-
      --model claude-opus-4-5-20251101
      --allowedTools "Bash(bun *)"
      --append-system-prompt "After making code changes, run tests."
      --debug
```

## Complete Working Example

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  issues:
    types: [opened, assigned, labeled]
  pull_request_review_comment:
    types: [created]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'issues' && (github.event.action == 'assigned' || github.event.action == 'labeled')) ||
      github.event_name == 'pull_request_review_comment'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      actions: read
    steps:
      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

          # Valid action inputs
          trigger_phrase: "@claude"
          use_sticky_comment: true
          additional_permissions: |
            actions: read

          # CLI flags via claude_args
          claude_args: >-
            --model claude-opus-4-5-20251101
            --allowedTools "Bash(bun *)"
            --append-system-prompt "Run 'bun run build' after making changes."
            --debug
```

## Valid Action Inputs Reference

These inputs can be used directly under `with:`:

### Authentication
| Input | Description |
|-------|-------------|
| `anthropic_api_key` | Anthropic API key |
| `claude_code_oauth_token` | Claude Code OAuth token (preferred) |
| `github_token` | GitHub token (defaults to `${{ github.token }}`) |

### Triggers
| Input | Description |
|-------|-------------|
| `trigger_phrase` | Phrase to trigger Claude (default: `@claude`) |
| `assignee_trigger` | Trigger when issue is assigned |
| `label_trigger` | Trigger when label is added |

### Branching
| Input | Description |
|-------|-------------|
| `base_branch` | Base branch for new branches |
| `branch_prefix` | Prefix for created branches |
| `branch_name_template` | Template for branch names |

### Access Control
| Input | Description |
|-------|-------------|
| `allowed_bots` | Bots allowed to trigger |
| `allowed_non_write_users` | Users without write access who can trigger |
| `include_comments_by_actor` | Include comments by specific actors |
| `exclude_comments_by_actor` | Exclude comments by specific actors |

### Configuration
| Input | Description |
|-------|-------------|
| `prompt` | Custom prompt for Claude |
| `settings` | JSON string or path to settings file |
| `claude_args` | **CLI flags go here** |
| `additional_permissions` | Extra GitHub permissions |

### Providers
| Input | Description |
|-------|-------------|
| `use_bedrock` | Use AWS Bedrock |
| `use_vertex` | Use Google Vertex AI |
| `use_foundry` | Use Foundry |

### Features
| Input | Description |
|-------|-------------|
| `use_sticky_comment` | Update same comment vs create new |
| `use_commit_signing` | Sign commits |

## CLI Flags for claude_args

Common CLI flags to pass via `claude_args`:

| Flag | Purpose | Example |
|------|---------|---------|
| `--model` | Set the model | `--model claude-opus-4-5-20251101` |
| `--allowedTools` | Allow specific tool patterns | `--allowedTools "Bash(bun *)"` |
| `--append-system-prompt` | Add to system prompt | `--append-system-prompt "Run tests"` |
| `--debug` | Enable debug logging | `--debug` |
| `--max-turns` | Limit agent turns | `--max-turns 10` |

## Pattern Matching for --allowedTools

The `--allowedTools` flag uses pattern matching:

```yaml
claude_args: >-
  --allowedTools "Bash(bun *)"      # Any bun command
  --allowedTools "Bash(npm run *)"  # Any npm run script
  --allowedTools "Bash(git add *)"  # Git add commands
```

### Pattern Syntax
- `*` matches any characters: `Bash(bun *)` matches `bun install`, `bun run build`, etc.
- Specific commands: `Bash(git commit -m *)` matches only git commit with message
- Multiple patterns: Use separate `--allowedTools` flags for each pattern

## Troubleshooting

### "Input not defined" Warnings
If you see warnings about undefined inputs, you're likely using CLI flags as direct inputs. Move them to `claude_args`.

### Model Not Changing
Ensure you're using `--model` inside `claude_args`, not as a direct input:

```yaml
# ❌ Wrong
model: claude-opus-4-5-20251101

# ✅ Correct
claude_args: --model claude-opus-4-5-20251101
```

### allowedTools Not Working
Check the pattern syntax and ensure it's in `claude_args`:

```yaml
# ❌ Wrong - not valid action input
allowed_tools: "Bash(bun *)"

# ✅ Correct - in claude_args with proper flag name
claude_args: --allowedTools "Bash(bun *)"
```

## Additional Resources

- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [claude-code-action Repository](https://github.com/anthropics/claude-code-action)
