# Permissions

## Read Operations

All read-only terminal operations are allowed without verification. This includes but is not limited to:
- ls, cat, head, tail, less, file, wc
- git status, git log, git diff, git branch
- find, grep, rg, fd
- tree, eza
- poetry show, poetry env info
- python -c (read-only inspection)
- Any command that only reads/inspects and does not modify state

## Write and Destructive Operations

Write and destructive operations require user verification before execution unless project-level rules explicitly grant broader permissions. This includes:
- File creation, modification, or deletion
- git commit, git push, git reset, git rebase
- Package installation or removal
- System configuration changes
