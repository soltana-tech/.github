---
name: audit
description: Run scoped code audits against coding principles, project conventions, and quality heuristics.
user-invocable: true
argument-hint: <scope|summary|scopes> [--skip-issues]
allowed-tools: Bash, Read, Grep, Glob, Task, AskUserQuestion, Write
---

# Audit Skill

Run scoped code audits that check for violations of coding principles (DRY, KISS, SOLID, etc.) and domain-specific quality criteria. Produces structured findings with severity levels and optional draft issue output.

## Project Configuration

Check for project-specific audit config at `<project>/.claude/audit/scopes.md`. If present:

- **H2 sections matching a global scope name** merge project-specific checks alongside generic checks for that scope.
- **H2 sections not matching any global scope** define new project-specific scopes with no generic fallback — the section content is the entire scope definition.

Also check `<project>/.claude/rules/` for project conventions (label taxonomy, issue types, coding principles) that inform finding classification.

## Actions

Interpret the user's arguments to determine which action to take.

### No arguments (list scopes)

List all available scopes with one-line descriptions. Include both global scopes and any project-defined scopes from `scopes.md`.

Format:

```
Available audit scopes:

  tests       Test quality, over-mocking, redundancy, tautological assertions
  logic       Dead code, unused exports, complexity, principle violations
  docs        Code-docs alignment, completeness, broken links, redundancy
  config      Over-parameterization, unused defaults, overlapping options
  structure   Circular deps, orphaned files, module boundary violations
  focus       Project adherence to stated purpose, scope creep
  <project>   <description from scopes.md>

Run: /audit <scope> to audit a specific scope
     /audit summary for a consolidated report across all scopes
```

### `<scope>` — Run a single scope audit

1. Validate the scope name exists (global or project-defined).
2. Load generic checks for the scope (see Global Scopes below).
3. If `scopes.md` has a matching H2 section, merge its checks and file scope.
4. If the scope is project-defined only, use the `scopes.md` definition as the entire scope.
5. Use `Glob`, `Grep`, `Read`, and `Task` (with Explore subagent) to inspect the codebase.
6. Produce findings in the output format described below.
7. Unless `--skip-issues` is specified, write WARNING and CRITICAL findings to `.claude/audit/draft-issues.md` (overwriting any existing content).

### `summary` — Consolidated report

Run all available scopes (global + project-defined). Display per-scope findings, then a summary table:

```
| Scope     | Critical | Warning | Info |
|-----------|----------|---------|------|
| tests     | 0        | 3       | 1    |
| logic     | 1        | 2       | 0    |
| ...       | ...      | ...     | ...  |
| **Total** | **1**    | **5**   | **1**|
```

Unless `--skip-issues` is specified, write all WARNING and CRITICAL findings to `.claude/audit/draft-issues.md`.

### `--skip-issues` flag

When present on any action, run the audit and display findings but do **not** write the draft issues file.

## Global Scopes

### `tests`

Test quality, over-mocking, redundancy, tautological assertions, third-party-only testing.

**Generic checks:**

- Tests that only verify third-party library calls succeed without testing project logic
- Excessive mocking that prevents testing real integration between components
- Tests with multiple mocks/spies that obscure what is actually being tested
- Redundant test cases that validate identical scenarios
- Tests that pass regardless of actual implementation (always-true assertions, tautological checks)
- Tests that work around bugs with mocks instead of testing the underlying code
- Tests for trivial getters/setters or pass-through functions
- Tests that do not fail when the tested code is intentionally broken
- Tests with unclear purpose or assertions that do not validate meaningful behavior
- Duplicate or near-duplicate tests that could be consolidated into parameterized test cases

### `logic`

Dead code, unused exports, complexity, SRP/LoD/DRY violations, type safety gaps.

**Generic checks:**

- Unreachable code paths and dead branches
- Exported symbols that are never imported elsewhere
- Functions exceeding reasonable cyclomatic complexity
- Modules or functions with multiple unrelated responsibilities (SRP violation)
- Deep property chain access across module boundaries (LoD violation)
- Copy-pasted logic that should be abstracted (DRY violation)
- Type safety gaps: `any` casts, missing null checks, unsafe type assertions
- Unused imports and variables
- Functions with excessive parameter counts (>4)
- Inconsistent error handling patterns

### `docs`

Code-documentation alignment, completeness, broken links, redundancy.

**Generic checks:**

- Documented features/APIs that do not exist in code
- Code features/APIs that lack documentation
- Examples in docs that do not work with the current API
- Parameter defaults in docs that do not match code defaults
- Broken internal links or anchors
- Duplicate content across documentation pages
- Overly technical or implementation-focused explanations (should be usage-focused)
- Missing practical examples or use cases
- Inconsistent terminology or naming between docs and code
- Orphaned or unreferenced documentation sections

### `config`

Over-parameterization, unused defaults, overlapping options, config complexity.

**Generic checks:**

- Parameters that are consistently set to the same value across the codebase
- Parameters that are rarely or never overridden from their defaults
- Parameters used in a single location only
- Excessive number of user-facing configuration options
- Parameters with unclear or overlapping purposes
- Mutually exclusive parameters that could be consolidated
- Parameters with complex interdependencies
- Missing validation for config values
- Config options that do not surface in any user-facing behavior

### `structure`

Circular dependencies, orphaned files, module boundary violations, inconsistent organization.

**Generic checks:**

- Circular import chains
- Files that are not imported or referenced anywhere
- Imports that cross module boundaries without going through public API
- Inconsistent directory structure or file organization patterns
- Mixed concerns within directories (e.g., utils mixed with components)
- Missing or inconsistent index/barrel files
- Build artifacts or generated files committed to source
- Inconsistent naming conventions across similar files

### `focus`

Project adherence to stated purpose, scope creep, project.md accuracy.

**Generic checks:**

- Features or code that fall outside the project's stated purpose (per `project.md` or README)
- `project.md` claims that do not match the current codebase state
- Scope creep: functionality that has drifted beyond the original project goals
- Competitive positioning claims that are inaccurate or unsupported
- Dead or abandoned features that no longer align with project direction
- Dependencies that suggest scope beyond the project's stated purpose

## Principle Mapping

Every finding must reference one of these principles:

| Principle | Finding Types |
|-----------|--------------|
| DRY | Duplicated logic, redundant tests, copy-paste code |
| KISS | Over-mocking, unnecessary abstraction, config complexity |
| SOLID (SRP) | Multi-purpose modules, mixed concerns |
| SOLID (DIP) | Concrete dependencies where abstractions should exist |
| LoD | Deep property chains, cross-boundary coupling |
| Fail Fast | Silent errors, tests that cannot fail |
| CoC | Naming inconsistencies, convention breaks |
| PoLA | API/docs mismatch, misleading names |
| Boy Scout | Stale code, outdated docs, dead imports |

Select the most specific principle that applies. If multiple principles apply, use the primary one.

## Output Format

### Individual findings

```
### [SEVERITY] Finding title
**Principle:** <principle> | **Scope:** <scope> | **File:** <path:line>
<Description of the finding.>
**Recommendation:** <Concrete action.>
```

Severities:

- **CRITICAL** — Bugs, security issues, or correctness problems that must be fixed.
- **WARNING** — Principle violations with real consequences (code smell, maintenance burden, user confusion).
- **INFO** — Improvement opportunities with no immediate consequence. Display-only; not written to draft issues.

### Scope summary (shown after all findings for a scope)

```
**<scope>**: X critical, Y warning, Z info
```

## Draft Issue File

When findings are produced and `--skip-issues` is not set, write WARNING and CRITICAL findings to `<project>/.claude/audit/draft-issues.md`, overwriting any previous content.

Format:

```markdown
<!-- Generated by /audit <scope|summary> — do not edit manually -->

## <Finding title>

- **Type:** Bug | Task
- **Labels:** <layer and size labels per project conventions>
- **Audit scope:** <scope>
- **Principle:** <principle>

### Description
<finding description>

### Proposed fix
<recommendation>

---
```

Rules for draft issue fields:

- **Type**: Use `Bug` for CRITICAL findings (bugs/security). Use `Task` for WARNING findings (principle violations, maintenance).
- **Labels**: Include the appropriate `layer:` label based on file type (`layer: scss`, `layer: typescript`, `layer: docs`). Estimate a `size:` label based on the scope of the fix (`size: S` for single-file, `size: M` for multi-file, `size: L` for cross-cutting, `size: XL` for architectural).
- **Audit scope**: The scope that produced the finding.
- **Principle**: The mapped principle from the table above.

If the project does not define a label taxonomy or issue types, omit the Type and Labels fields and write only the title, audit scope, principle, description, and proposed fix.
