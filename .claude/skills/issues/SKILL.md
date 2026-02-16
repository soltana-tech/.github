---
name: issues
description: List, view, create, add, close, triage, and search GitHub Issues for the current repository.
user-invocable: true
argument-hint: [action] [number|term]
allowed-tools: Bash, Read, Grep, Glob, Task, EnterPlanMode, AskUserQuestion
---

# GitHub Issues Skill

Manage GitHub Issues for the current repository via the `gh` CLI.

Check the current repository's rules (e.g., `.claude/rules/`) for project-specific conventions: label taxonomy, issue types, body templates, and priority heuristics. Defer to those conventions when they exist. When no repo-specific conventions are defined, use the generic defaults described below.

## Detecting Issue Types

Some organizations use GitHub Issue Types (org-level). To check, run:

```bash
gh api orgs/<org>/issue-types
```

If the endpoint returns types, use GraphQL to include them in queries:

```bash
gh api graphql -f query='{
  repository(owner: "<owner>", name: "<repo>") {
    issues(first: 50, states: OPEN, orderBy: {field: CREATED_AT, direction: DESC}) {
      nodes { number title state issueType { name } labels(first: 10) { nodes { name } } }
    }
  }
}'
```

If the endpoint returns 404 or empty, issue types are not in use — omit them from output.

## Actions

Interpret the user's arguments to determine which action to take.

### `list` (default if no arguments)

List open issues.

```bash
gh issue list --state open --json number,title,labels,state -L 50
```

If the repository uses Issue Types, use GraphQL instead (see above).

Format the output as a table: `#number — title [type] [labels]`. Omit the type column if not in use.

### `view <number>`

Show full detail for a single issue.

```bash
gh issue view <number> --json number,title,body,labels,state,comments
```

If the repository uses Issue Types, also fetch the type via GraphQL.

Display the issue title, type (if applicable), state, labels, body, and comments.

### `plan <number>`

Fetch the full issue detail — including comments — then enter plan mode to formulate an implementation plan.

```bash
gh issue view <number> --json number,title,body,labels,state,comments
```

If the repository uses Issue Types, also fetch the type via GraphQL.

Display the issue title, type (if applicable), state, labels, body, and comments before entering plan mode so full context is visible.

After fetching, call `EnterPlanMode`. The plan should:
- Identify all files that need to change
- Describe each change concretely
- List tests to add or update
- Note dependencies on other issues
- Define verification steps

### `create`

Interactively create a new issue. Check repo rules for:
- Available issue types (GitHub Issue Types)
- Label categories and valid values
- Body template structure

Use `AskUserQuestion` to gather the required fields. At minimum, collect:
- Title
- Description

If repo rules define issue types, also gather the type. If repo rules define label categories, also gather the relevant labels. If repo rules define a body template, use it.

When no repo rules exist, gather title and description only, and ask about labels freeform.

Create the issue:

```bash
gh issue create --title "<title>" --label "<labels>" --body "<body>"
```

If an issue type was selected, set it after creation via GraphQL:

```bash
# Get the issue node ID
gh issue view <number> --json id --jq .id

# Get the issue type node ID
gh api orgs/<org>/issue-types --jq '.[] | select(.name == "<type>") | .node_id'

# Set the type
gh api graphql -f query='
  mutation {
    updateIssue(input: {id: "<issue_node_id>", issueTypeId: "<type_node_id>"}) {
      issue { number issueType { name } }
    }
  }'
```

### `add`

Batch-create issues from audit draft file.

Read `.claude/audit/draft-issues.md`. If the file does not exist or is empty, report that no
draft issues are pending and exit.

Parse each H2 section as a draft issue. For each draft, present the title, type, labels, and
description to the user via `AskUserQuestion` with options: Create, Skip, Edit.

- **Create**: Create the issue using `gh issue create` with the repo's conventions (issue types,
  label taxonomy, body template from `.claude/rules/`). Set the issue type via GraphQL if the
  org uses Issue Types.
- **Skip**: Do not create this issue. Move to the next draft.
- **Edit**: Use `AskUserQuestion` to let the user modify the title, type, labels, or description
  before creating.

After processing all drafts, delete `.claude/audit/draft-issues.md`.

Display a summary: N created, N skipped, N edited.

### `close <number>`

Close an issue. Use `AskUserQuestion` to get a closing comment summarizing what was done, then:

```bash
gh issue close <number> --reason completed --comment "<summary>"
```

### `status`

Show issue counts by state.

```bash
gh issue list --state all --json state -L 500
```

Count and display: open, closed, total. If Issue Types are in use, also break down by type.

### `next`

Recommend the next issue to work on.

Check repo rules for priority heuristics. If defined, apply them. If the heuristics reference issue types, use GraphQL to fetch type data alongside labels.

When no repo-specific heuristics exist, use this default ranking:

1. Exclude blocked issues (any label containing "blocked")
2. Lower issue number first (older issues)

Present the top 5 candidates as a numbered list with labels and type (if applicable). Briefly note why each ranks where it does.

### `triage <number>`

Determine whether an issue is still relevant to the current state of the codebase.

```bash
gh issue view <number> --json number,title,body,labels,state,comments
```

After fetching the issue, inspect the codebase using `Grep`, `Glob`, and `Read` to evaluate:

1. **Existence** — Do the files, components, or tokens mentioned in the issue still exist?
2. **Current state** — Does the described bug, gap, or missing feature still apply, or has subsequent work already addressed it?
3. **Staleness** — Has the area of code changed significantly since the issue was filed, potentially invalidating its assumptions?

Produce a verdict with one of three labels:

- **Relevant** — The issue accurately describes a current problem or gap. No action needed on the issue itself.
- **Stale** — The codebase has changed enough that the issue's description no longer matches reality. Recommend updating or closing.
- **Resolved** — The requested change or fix already exists in the codebase. Recommend closing.

Include a short explanation citing specific files or code that supports the verdict. If the verdict is **Stale** or **Resolved**, suggest an appropriate closing comment.

### `search <term>`

Search issues by keyword.

```bash
gh issue list --search "<term>" --state all --json number,title,labels,state -L 50
```

Format as a table: `#number — title [labels] (state)`.
