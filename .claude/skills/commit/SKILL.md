---
name: commit
description: Stage and commit changes with a well-formed commit message.
disable-model-invocation: true
allowed-tools: Bash
---

# Commit

Stage and commit the current changes following strict commit message rules.

## Commit Message Rules

- First line MUST be under 50 characters
- Use imperative mood (e.g., "Fix bug" not "Fixed bug")
- Never mention Claude, Claude Code, AI, or any AI tool
- Never include `Co-Authored-By` lines
- If the project has a `.claude/rules/` directory or project-level CLAUDE.md, check for additional commit message conventions (e.g., issue prefixes, ticket formats, scope tags) and apply them on top of the rules above

## Process

1. Run these in parallel to understand the current state:
   - `git status` (never use `-uall`)
   - `git diff` and `git diff --cached` to see staged and unstaged changes
   - `git log --oneline -5` to see recent commit style

2. Analyze all changes and draft a commit message:
   - Summarize the nature of the change (new feature, bug fix, refactor, etc.)
   - Use "add" for wholly new features, "update" for enhancements, "fix" for bug fixes
   - Do not commit files that likely contain secrets (.env, credentials, etc.)
   - Focus the message on "why" rather than "what"

3. Present the proposed commit message and list of files to be staged. Wait for user approval before proceeding.

4. After approval, stage the relevant files by name (never use `git add -A` or `git add .`) and create the commit. Use a HEREDOC to pass the message:

```
git add <files> && git commit -m "$(cat <<'EOF'
Commit message here
EOF
)"
```

5. If a pre-commit hook fails, fix the issue, re-stage, and create a NEW commit. Never amend.

6. Run `git status` after the commit to verify success.

## Arguments

If `$ARGUMENTS` is provided, use it as guidance for the commit message content.
