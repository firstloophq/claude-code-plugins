---
name: ship-and-monitor
description: Commit, push, open a PR, and monitor CI/CD in a loop. Use when the user says "ship it", "commit push and watch CI", or wants to open a PR and monitor checks. Auto-fixes obvious CI failures and stops to ask for non-obvious ones.
---

# Ship and Monitor

Commits changes, pushes, opens a PR, and monitors CI/CD checks — auto-fixing obvious failures and asking the user about non-obvious ones.

## Workflow

### Step 1: Confirm Branch

Before committing, verify the current branch is correct. In long conversations that span multiple branches, never add commits to a branch that was already merged.

```bash
git status
git log --oneline -5
git branch --show-current
```

**Check for these red flags:**
- Current branch has already been merged (check with `gh pr list --state merged --head <branch>`)
- Branch name doesn't match the work that was just done
- Unexpected uncommitted changes from unrelated work

If anything looks off, **stop and confirm with the user** using the `AskUserQuestion` tool before proceeding.

### Step 2: Commit and Push

1. **Stage relevant files only** — never use `git add -A` or `git add .`. Review changes and stage specific files:
   ```bash
   git diff --name-only
   git diff --staged --name-only
   ```
2. **Create a descriptive commit** — summarize the "why", not just the "what":
   ```bash
   git add <specific-files>
   git commit -m "<descriptive message>"
   ```
3. **Push with upstream tracking:**
   ```bash
   git push -u origin <branch-name>
   ```

### Step 3: Open PR

Open a pull request targeting `develop` by default, or a user-specified branch.

```bash
gh pr create --title "<concise title>" --body "$(cat <<'EOF'
## Summary
<bullet points describing the changes>

## Test plan
- [ ] CI checks pass
- [ ] <relevant test items>

Generated with [Claude Code](https://claude.ai/code)
EOF
)"
```

If the user specified a target branch, use `--base <branch>`. Otherwise default to `develop`. If `develop` doesn't exist, fall back to `main`.

**Show the PR URL to the user after creation.**

### Step 4: Monitor CI/CD with `/loop`

Start monitoring CI/CD checks at 1-minute intervals:

```
/loop 1m check CI status for PR
```

Each loop iteration should:

1. **Check PR status:**
   ```bash
   gh pr checks <pr-number> --watch --fail-fast
   ```
   Or poll with:
   ```bash
   gh pr checks <pr-number>
   ```

2. **On all checks passing:** report success and stop the loop.

3. **On failure:** classify the failure and either auto-fix or stop to ask.

---

## Failure Classification

### Auto-fix (obvious failures)

Fix these automatically, commit, push, and continue monitoring:

- **TypeScript/build errors** — type mismatches, missing imports, unused variables
- **Lint/format violations** — ESLint, Prettier, Biome, etc.
- **Test failures clearly caused by our changes** — wrong assertion values, missing mock updates, snapshot mismatches
- **Missing documented environment variables** — variables referenced but not set in CI config

**Auto-fix workflow:**
1. Read the CI log to identify the error
2. Fix the issue locally
3. Stage only the changed files
4. Commit with a message like `fix: resolve lint error in <file>` or `fix: update test assertion for <change>`
5. Push
6. Continue monitoring

### Stop and ask (non-obvious failures)

Stop the loop and report these to the user with full context:

- **Test failures in code we didn't touch** — may indicate a pre-existing issue or an unexpected side effect
- **Infrastructure/deployment failures** — Docker build errors, Terraform issues, cloud provider errors
- **Flaky or intermittent test failures** — tests that passed locally or in previous runs
- **Permission/auth errors** — missing secrets, expired tokens, insufficient permissions
- **Fixes that would require architectural changes** — anything beyond the scope of the original intent

**When stopping, provide:**
- The specific check that failed
- Relevant log output (trimmed to the useful parts)
- Your assessment of the likely cause
- Suggested next steps

---

## Branch Safety Rules

- **Always verify the current branch** before committing
- **Check if the branch was already merged** — in long conversations, the user may have moved on
- **Never blindly commit** to whatever branch happens to be checked out
- **If in doubt, ask** — a 10-second confirmation is better than committing to the wrong branch

## Examples

**User says:** "ship it"
→ Confirm branch → stage & commit → push → open PR → start `/loop 1m` monitoring

**User says:** "commit push and watch CI"
→ Same flow

**User says:** "ship it to main"
→ Same flow but target `main` as the PR base branch

**CI lint fails:**
→ Read log → fix lint error → commit → push → continue monitoring

**CI test fails in unrelated code:**
→ Stop loop → report failure details → ask user for guidance

**Branch was already merged:**
→ Stop → tell the user → ask what branch to use
