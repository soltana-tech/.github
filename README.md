# .github

Org-level defaults for soltana-tech repositories.

- **Issue templates** — standardized task format
- **PR template** — summary + test plan checklist

## Claude Code Configuration

The `.claude/` directory contains the shared Claude Code configuration for all soltana-tech repositories. This serves as the **global layer** — rules, skills, and conventions that apply across every project in the org.

### Structure

```
.claude/
├── rules/              # Auto-loaded into every Claude Code session
│   ├── agent-behavior.md   # Coding principles (DRY, KISS, SOLID, etc.)
│   ├── git.md              # Commit message and git safety rules
│   └── permissions.md      # Read vs. write operation guardrails
└── skills/             # Slash commands available in all projects
    ├── audit/SKILL.md      # /audit — scoped code quality audits
    ├── commit/SKILL.md     # /commit — guided staging and commit flow
    └── issues/SKILL.md     # /issues — GitHub Issues management
```

### How Global and Project Config Interact

Claude Code loads configuration from two layers, merged at runtime:

| Layer | Location | Loaded when |
|-------|----------|-------------|
| **Global** | `~/.claude/rules/`, `~/.claude/skills/` | Every session, every project |
| **Project** | `<repo>/.claude/rules/`, `<repo>/.claude/skills/`, `<repo>/.claude/audit/` | Only in that repo's sessions |

**Rules** (`.claude/rules/*.md`) auto-load into context at session start. Global rules apply everywhere. Project rules add to them — they do not replace global rules. Both sets are visible simultaneously.

**Skills** (`.claude/skills/<name>/SKILL.md`) register slash commands. Global skills are available in all projects. Project skills are only available in that repo. If a project defines a skill with the same name as a global skill, the project skill takes precedence.

**Audit scopes** (`.claude/audit/scopes.md`) are project-specific config consumed by the global `/audit` skill. They live outside `.claude/rules/` intentionally — audit scope definitions are large and only needed when `/audit` is invoked, so they should not auto-load into every session. A project's `scopes.md` can extend the 6 built-in global scopes with project-specific checks and file scopes, or define entirely new scopes.

### Precedence Summary

1. Global rules + project rules both load (additive)
2. Project skills override global skills of the same name
3. Project `settings.local.json` controls per-repo tool permissions (gitignored)
4. Audit scopes are lazy-loaded by `/audit` only when invoked
