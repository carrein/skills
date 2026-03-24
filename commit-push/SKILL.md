---
name: commit-push
description: >
  Stage all changes, split into logical commits, and push to the current
  branch. Use when the user says "commit", "push", "commit and push",
  "ship it", "save changes", or "cp".
disable-model-invocation: true
---

# Commit & Push

You are committing and pushing changes for a codebase. Your job is to
stage everything, split changes into logical commits, generate accurate
commit messages that match the repo's existing style, run checks, and
push to the current branch.

---

## Step 1: Analyze Changes

- Run `git status` and `git diff` to understand all unstaged and
  staged changes
- Read the recent commit history (`git log --oneline -20`) to
  detect the repo's commit message style
- Identify the current branch

---

## Step 2: Detect Commit Message Style

The commit messages must match the existing style in the repo.
Examine the last 20 commits and detect:

- **Format:** Does the repo use conventional prefixes (feat:, fix:,
  docs:, chore:, refactor:, quality:, etc.), plain imperative
  messages, or a mix of both?
- **Casing:** Lowercase after prefix? Capitalized for plain messages?
- **Subject line style:** Imperative mood? Max length?
- **Body style:** Does the repo use detailed bodies? How are they
  structured — prose, bullet lists, grouped by area?
- **Trailers:** Any consistent patterns like Co-Authored-By,
  Signed-off-by, etc.?

Match whatever the repo does. Do not impose a new convention.

---

## Step 3: Split Changes Logically

Do NOT bundle everything into one commit. Group changes into logical
units based on what they do, not where they are. Examples of good
splits:

- UI changes separate from backend changes
- Bug fixes separate from feature additions
- Refactors separate from new functionality
- Documentation updates separate from code changes
- Config/dependency changes separate from source changes

Each commit should be a coherent, self-contained unit that could be
reverted independently without breaking other commits in the set.

If all changes are genuinely part of one logical unit, a single
commit is fine. Don't split artificially.

---

## Step 4: Pre-Commit Checks

Before committing, run whatever checks are available in the project:

1. **Linter** — detect and run the project's linter (eslint, dart
   analyze, pylint, clippy, etc.). If lint fails, fix the issues
   first, include the fixes in the relevant commit.
2. **Tests** — detect and run the project's test suite (jest, pytest,
   flutter test, go test, cargo test, etc.). If tests fail:
   - If the failure is caused by your staged changes, fix it
   - If the failure is pre-existing (fails on a clean checkout too),
     note it in the terminal output and proceed
   - Do NOT commit code that introduces new test failures

If no linter or test runner is detected, note this in the output
and proceed.

---

## Step 5: Generate Commit Messages & Commit

For each logical commit:

1. Stage the relevant files (`git add` per group)
2. Generate a commit message matching the detected style:
   - **Subject line:** concise, matches the format and casing
     conventions found in Step 2
   - **Body:** include a detailed summary of what changed and why,
     matching the body style of existing commits. If the repo uses
     bodies, include one. If it doesn't, skip it.
3. **Co-Authored-By:** Always include a Co-Authored-By trailer when
   Claude is generating the commit. Use the format found in the
   repo's existing commits. If no prior format exists, use:
   `Co-Authored-By: Claude <noreply@anthropic.com>`
4. Commit

Do not ask for approval on commit messages. Auto-generate and commit.
The user will amend if needed.

---

## Step 6: Push

- Push to the current branch: `git push`
- If the branch has no upstream, set it: `git push -u origin HEAD`
- If the push is rejected (behind remote), pull with rebase first:
  `git pull --rebase` then push again
- If rebase conflicts arise, stop and report — do not auto-resolve
  merge conflicts

---

## Step 7: Summary

Print a summary to the terminal:
```
═══════════════════════════════════════
  COMMIT & PUSH SUMMARY
  Branch: [branch name]
  Remote: [remote/branch]
═══════════════════════════════════════

  Lint:  [passed | N issues fixed | not available]
  Tests: [passed | N pre-existing failures | not available]

  Commits pushed:
  1. [short hash] [subject line]
  2. [short hash] [subject line]
  3. [short hash] [subject line]

  Files changed: [n]  |  Insertions: [n]  |  Deletions: [n]
═══════════════════════════════════════
```

---

## Principles

- **Match the repo, not a standard.** The existing commit style is
  the correct one. Detect it and follow it.
- **Logical splits over file splits.** Group by what the change does,
  not by file path or language.
- **Never push broken code.** If lint or tests fail because of your
  changes, fix before committing.
- **Never auto-resolve conflicts.** If there's a merge conflict,
  stop and let the user handle it.
- **Speed over ceremony.** This is a fast-flow skill. No approval
  steps, no interactive prompts. Stage, commit, push, report.
