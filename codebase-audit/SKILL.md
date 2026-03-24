---
name: codebase-audit
description: >
  Assess codebase quality and realign conventions, documentation, and patterns
  that have drifted across multiple AI sessions. Use when the user says
  "code quality", "assess quality", "realign codebase", "convention check",
  "codebase audit", or "align conventions".
disable-model-invocation: true
---

# Code Quality Assessment & Realignment

You are auditing a codebase that has been primarily written and maintained
by multiple AI sessions over time. Different sessions may have introduced
inconsistent conventions, divergent documentation, stale comments, and
conflicting patterns. Your job is to detect this drift, report it clearly,
and — with approval — bring the codebase back into alignment so the next
AI session inherits a coherent, understandable project.

---

## Phase 0: Scope & Plan

### Detect Context

- Identify the primary language(s) and framework(s) in use
- Read existing config files (linters, formatters, tsconfig, pubspec,
  pyproject, etc.) to understand the project's declared standards
- Scan folder structure to understand the architecture
- Read any existing documentation (README, ARCHITECTURE.md, CONTRIBUTING,
  inline doc comments) to understand what the project claims about itself
- Identify the dominant conventions already in use — these are the
  baseline to align toward, not an external standard

The goal is to align the codebase with ITSELF, not impose external
opinions. The most common pattern wins. Outliers get flagged.

### Exclusions

Always skip the following. Do not read, assess, or modify these:

- Dependency directories: node_modules/, .dart_tool/, **pycache**/,
  venv/, .venv/, vendor/
- Build output: build/, dist/, .next/, out/, .build/
- Generated code: _.g.dart, _.freezed.dart, _.gen.ts, _.generated.\*,
  and any file with a "DO NOT EDIT" or "GENERATED" header
- Lock files: package-lock.json, pubspec.lock, yarn.lock, pnpm-lock.yaml,
  poetry.lock, Gemfile.lock, go.sum
- Assets: images, fonts, icons, SVGs, audio, video files
- Anything matched by the project's .gitignore

When in doubt whether a file is generated, check for a generated header
comment in the first 5 lines. If present, skip it.

### Size Assessment & Agent Strategy

- Count total source files and lines (after exclusions)
- If the repo has **fewer than 500 source files**: proceed as a single agent
- If the repo has **500+ source files**: use multi-agent strategy:
  1. Identify logical modules (top-level directories, packages, or
     workspace members)
  2. Spawn one sub-agent per module — each runs Phase 1 and Phase 2
     independently against its module
  3. This lead agent collects all sub-agent findings, deduplicates
     cross-module issues, and produces the unified report in Phase 3
  4. Cross-cutting concerns (shared utilities used across modules,
     inconsistent patterns BETWEEN modules) are assessed by the lead
     agent after sub-agents complete

Print the plan before starting:

```
Scope: [n] source files, [n] lines across [n] modules
Strategy: [single agent | multi-agent with N sub-agents]
Modules: [list of module names]
Starting assessment...
```

---

## Phase 1: Assess

Evaluate source code across six categories, in this priority order:

### 1. Performance (highest priority)

- Unnecessary re-renders, redundant computations, N+1 patterns
- Synchronous operations that should be async
- Large imports where tree-shakeable or lighter alternatives exist
- Missing caching, memoization, or indexing where obviously beneficial
- Resource leaks (unclosed streams, connections, listeners)

### 2. Error Handling

- Swallowed errors (empty catch blocks, ignored futures)
- Inconsistent error handling strategies across the codebase
- Missing error boundaries or fallback behavior
- Generic error messages that lose context
- Unhandled edge cases (null, empty, malformed input)

### 3. Test Coverage

- Critical paths without test coverage
- Tests that exist but are stale (testing old behavior)
- Inconsistent testing patterns across similar modules
- Missing edge case tests for error paths

### 4. Type Safety

- Weak or missing types (any, dynamic, Object, untyped maps)
- Inconsistent type usage across similar functions
- Type assertions or casts that bypass safety
- Missing null safety or optional handling

### 5. Modularity & Reuse

- Duplicated logic that should be extracted into shared utilities
- Large files doing too many things (separation of concerns)
- Functions with multiple responsibilities
- Tightly coupled modules that should be independent
- Logic mixed into UI or data layers where it doesn't belong

### 6. Readability & Naming

- Inconsistent naming conventions (camelCase in some files,
  snake_case in others for the same language)
- Ambiguous or misleading function/variable names
- Inconsistent file/folder naming patterns
- Dead code, unused imports, commented-out blocks

---

## Phase 2: Documentation Accuracy Check

This is NOT about generating new documentation. Check that what EXISTS
is still true:

- Do inline comments accurately describe what the code currently does?
- Do docstrings/JSDoc/dartdoc match current function signatures
  and behavior?
- Does the README reflect the current state of the project (setup
  steps, architecture, dependencies)?
- Do any existing architecture or design docs match reality?
- Are TODO/FIXME comments still relevant or have they been resolved?

Flag any documentation that has drifted from the code it describes.

---

## Phase 3: Report

Print a traffic light report to the terminal:

```
══════════════════════════════════════════════════
  CODE QUALITY REPORT — [project name]
  Language: [detected]  |  Framework: [detected]
  Files scanned: [n]    |  Lines: [n]
  Strategy: [single | multi-agent, N modules]
══════════════════════════════════════════════════

  1. 🔴 Performance          [one-line summary of worst issue]
  2. 🟡 Error Handling       [one-line summary]
  3. 🟢 Test Coverage        [one-line summary]
  4. 🟢 Type Safety          [one-line summary]
  5. 🟡 Modularity & Reuse   [one-line summary]
  6. 🟢 Readability & Naming [one-line summary]

  📄 Documentation Accuracy: [X] issues found

══════════════════════════════════════════════════
```

Traffic light criteria:

- 🟢 Green: Consistent across the codebase, no significant issues
- 🟡 Yellow: Some drift or minor issues found, worth addressing
- 🔴 Red: Significant inconsistency or problems, should fix

---

## Phase 4: Present Findings for Approval

Group ALL findings across all categories by severity:

### 🔴 Critical

For each issue:

- **File(s):** [paths]
- **Category:** [which of the 6]
- **Issue:** [what's wrong]
- **Proposed fix:** [what the change would be, briefly]

### 🟡 Moderate

Same format.

### 🟢 Minor

Same format.

Then ask:
**"Which groups should I fix? (e.g., 'all red', 'red and yellow', 'all', or 'none')"**

Wait for explicit approval before proceeding. Do not fix anything
without the user's go-ahead.

---

## Phase 5: Apply Approved Changes

For each approved group:

1. Make changes that align the codebase with its own dominant conventions
2. After each code change, check if any inline comments or docstrings
   near the changed code are now inaccurate — if so, update them
3. Ensure existing documentation still reflects reality after changes
4. Run any available linter, formatter, or test suite to verify
   nothing is broken
5. Commit per severity group with a clear message:
   `quality: fix [critical|moderate|minor] issues — [brief summary]`

If a change breaks tests or the build, revert that specific change,
report it as unresolvable, and continue with the rest.

---

## Principles

- **Align with the codebase, not with you.** The most common existing
  pattern is the correct one. Do not introduce new conventions.
- **Convention convergence over preference.** If 80% of files use one
  pattern, align the other 20% to match.
- **AI-readability is a first-class goal.** Every change should make it
  easier for the next AI session to understand the project. Clear names,
  small focused functions, explicit types, accurate comments.
- **Never change behavior.** This is alignment, not feature work.
  Inputs and outputs of every function must stay the same.
- **When in doubt, flag it — don't fix it.** If a pattern might be
  intentional, report it in the findings and let the user decide.
- **Language-agnostic assessment.** Apply the standards and idioms
  appropriate to whatever language and framework the project uses.
  Do not impose conventions from one language onto another.
- **Do not skip files to save time.** Assess every source file, even
  if the repo is large. Thoroughness is more important than speed.
