---
name: release
description: >
  Create a GitHub release with auto-generated version bump, release notes,
  and optional build artifacts. Use when the user says "release", "create
  release", "cut a release", "ship a release", "tag a release", or "new
  version".
disable-model-invocation: true
---

# GitHub Release

You are creating a GitHub release for a codebase. Your job is to detect
the versioning scheme, bump the version, generate release notes from
commits, run checks, tag, push, and create the release on GitHub.

---

## Step 0: Prerequisites

Before starting, verify required tools and repo state:

1. Run `gh --version` to check if the GitHub CLI is installed
   - If not installed: print "❌ gh CLI is required but not found.
     Install it: https://cli.github.com" and stop immediately
2. Run `gh auth status` to check authentication
   - If not authenticated: print "❌ gh CLI is not authenticated.
     Run `gh auth login` first." and stop immediately
3. Verify the current directory is a Git repo with a remote
   - If not: print "❌ Not a Git repository with a remote." and
     stop immediately
4. Run `git status --porcelain` to check for uncommitted changes
   - If the working tree is dirty (any output): print
     "❌ Working tree is not clean. Commit or stash your changes
     before releasing." and stop immediately

Do not proceed to any other step if prerequisites fail.

---

## Step 1: Detect Versioning Scheme

- Find the latest Git tag: `git describe --tags --abbrev=0` or
  `git tag --sort=-v:refname | head -1`
- Examine the tag history (`git tag --sort=-v:refname | head -10`)
  to detect the pattern:
  - **Simple incremental:** v1.00, v1.01, v1.02 → bump by +0.01
  - **Semver:** v1.2.3 → bump based on commit types
  - **CalVer:** 2026.03.25 → bump based on date
  - **Other:** match whatever pattern exists
- If no tags exist, check version files (package.json, pubspec.yaml,
  Cargo.toml, pyproject.toml, etc.) for the current version
- If no version exists anywhere, ask the user for a starting version

Also detect the tag style:
- Run `git cat-file -t <latest-tag>` to determine if existing tags
  are **lightweight** or **annotated**
- Match whatever the repo already uses

Match the existing scheme. Do not switch conventions.

---

## Step 2: Determine Version Bump

Collect all commits since the last release tag:
`git log <last-tag>..HEAD --oneline`

**If there are zero commits since the last tag**, stop immediately:
print "ℹ️ No commits since last release ([last tag]). Nothing to
release." and exit. Do not proceed.

**For semver repos**, determine bump from commits:
- Any commit with `feat:` or a new feature verb (Add, Implement,
  Introduce) → **minor** bump
- Only `fix:`, `docs:`, `chore:`, `refactor:`, `quality:`, or
  maintenance verbs (Fix, Update, Remove, Clean) → **patch** bump
- Any commit mentioning BREAKING CHANGE in the body → **major** bump

**For simple incremental repos** (+0.01 style):
- Always increment by the same step detected in tag history

**For CalVer repos:**
- Use today's date in the detected format

---

## Step 3: Confirm Before Proceeding

Print the release plan and wait for explicit approval:
```
═══════════════════════════════════════
  RELEASE PLAN
═══════════════════════════════════════

  Current version:  [old version]
  Proposed version: [new version]
  Bump type:        [major | minor | patch | incremental | calver]
  Bump reason:      [e.g., "detected feat/Add commits" or
                     "fixes and maintenance only" or
                     "incremental +0.01"]
  Branch:           [current branch]
  Tag style:        [lightweight | annotated]

  Commits to include ([n]):
  - [commit subject 1]
  - [commit subject 2]
  - [commit subject 3]
  ...

═══════════════════════════════════════
```

Then ask: **"Proceed with this release? (yes/no)"**

Do not continue unless the user explicitly confirms.

---

## Step 4: Pre-Release Checks

### Discover Packages

Auto-discover all packages and subprojects in the repo. Look for:
- Subdirectories with their own pubspec.yaml, package.json,
  Cargo.toml, pyproject.toml, go.mod, build.gradle, etc.
- Monorepo workspace definitions (pnpm-workspace.yaml, Cargo
  workspace members, etc.)
- The root project itself

**Skip generated packages.** For each discovered package, sample
the first 5 lines of up to 10 source files. If the majority contain
generated headers ("DO NOT EDIT", "GENERATED", "@generated",
"AUTO-GENERATED", "This file is generated"), skip the entire package.
Exception: if the package has a `test/` directory with non-generated
test files, include it anyway.

Print which packages are included and which are skipped:
```
  Packages:
  ✅ memoka_server/   — included
  ✅ memoka_flutter/  — included
  ⏭️  memoka_client/  — skipped (generated code)
```

### Run Checks

For each included package, run all available checks:

1. **Formatter** — detect and run the project's formatter with a
   check flag (dart format --set-exit-if-changed ., prettier --check,
   black --check, rustfmt --check, gofmt -l, etc.)
2. **Linter** — detect and run the project's linter (dart analyze,
   eslint, pylint, clippy, golangci-lint, etc.)
3. **Tests** — detect and run the project's test suite (dart test,
   flutter test, jest, pytest, cargo test, go test, etc.)

**If any check fails in any included package, stop the release
immediately and report what failed. No exceptions. No overrides.**

Print check results per package:
```
  Checks:
  ✅ memoka_server/   format ✓  lint ✓  tests ✓
  ✅ memoka_flutter/  format ✓  lint ✓  tests ✓
```

### Build & Artifacts

Check for CI release workflows:
- Look for `.github/workflows/release.yml`, `.github/workflows/deploy.yml`,
  or any workflow triggered by `on: release` or `on: push: tags:`
- If a release workflow exists: **skip local build and artifact
  generation entirely**. CI handles it.
- If no release workflow exists: run any detected build script to
  confirm the project compiles, and collect artifacts for attachment.

---

## Step 5: Update Version Files (conditional)

This step only applies if version files track the release version.

- Scan for version files (package.json, pubspec.yaml, Cargo.toml,
  pyproject.toml, build.gradle, etc.)
- For each file found, compare its current version value against
  the **previous Git tag version**
- **Only update files where the current value matches the old tag
  version.** If a file's version does NOT match the old tag (e.g.,
  pubspec.yaml says `0.0.0+1` but the last tag is `v0.5.0`), leave
  it untouched — the project intentionally manages that version
  differently.
- If no files match, skip this step entirely. The Git tag IS the
  version.

If files were updated, commit the changes with a message matching
the repo's commit style. If no prior version bump commits exist,
use: `chore: bump to [version]`

Include Co-Authored-By trailer since Claude is generating this
commit. Use the format found in the repo's existing commits.
If no prior format exists, use:
`Co-Authored-By: Claude <noreply@anthropic.com>`

---

## Step 6: Tag & Push

1. Create a tag on HEAD (or the bump commit if Step 5 produced one),
   matching the detected tag style:
   - **Lightweight:** `git tag [version]`
   - **Annotated:** `git tag -a [version] -m "Release [version]"`
2. Push any commits and the tag:
   `git push && git push --tags`
3. If push is rejected, pull with rebase first:
   `git pull --rebase && git push && git push --tags`
4. If rebase conflicts arise, stop and report — do not auto-resolve

---

## Step 7: Generate Release Notes

Collect all commits between the previous tag and the new tag.
Group them by type, stripping conventional commit prefixes from
the line since the section header already communicates the category:
```
## What's New
- Overhaul media upload pipeline and image caching architecture
- Add video thumbnails to upload dialog masonry grid

## Fixes
- Resolve crash on Android resume with post-frame callback
- Downloads lost on channel switch with persistent DownloadTracker

## Maintenance
- Codebase audit — fix critical, moderate, and minor issues
- Add TechDebt.md for tracked audit follow-ups
```

**Categorization rules:**
- Prefixed commits → categorize by prefix, strip the prefix:
  - `feat: add search` → "Add search" under What's New
  - `fix: resolve crash` → "Resolve crash" under Fixes
  - `docs:`, `chore:`, `refactor:`, `quality:` → Maintenance
- Plain imperative commits → categorize by leading verb:
  - Add, Implement, Introduce, Overhaul, Create → What's New
  - Fix, Resolve, Correct → Fixes
  - Everything else → Maintenance
- Capitalize the first letter after stripping the prefix
- Omit the version bump commit itself from the notes
- Omit merge commits
- If a category has no commits, omit that section entirely

Write the notes to a temporary file for use in Step 8.

---

## Step 8: Create GitHub Release

Using the `gh` CLI with a notes file to handle multiline content:
```
gh release create [version] \
  --title "[version]" \
  --notes-file [temp-notes-file] \
  [artifact paths if any]
```

If Step 4 determined that CI handles builds and no local artifacts
were collected, do not attach any files.

Clean up the temporary notes file after the release is created.

---

## Step 9: Summary

Print a summary to the terminal:
```
═══════════════════════════════════════
  RELEASE SUMMARY
  Version: [version]
  Tag: [tag name] ([lightweight | annotated])
  Branch: [branch name]
═══════════════════════════════════════

  Packages checked:
  [package 1]:  format ✓  lint ✓  tests ✓
  [package 2]:  format ✓  lint ✓  tests ✓
  [package 3]:  ⏭️ skipped (generated)

  Build: [skipped — CI handles | passed | not available]

  Release notes:
  - [n] new features
  - [n] fixes
  - [n] maintenance

  Artifacts: [list | "none — CI handles" | "none"]

  🔗 [GitHub release URL]
═══════════════════════════════════════
```

---

## Principles

- **Match the repo, not a standard.** Detect and follow whatever
  versioning scheme, tag style, and commit format already exists.
- **Never release broken code.** If any formatter, linter, or test
  fails in any included package, stop immediately. No exceptions.
- **Don't fight CI.** If a release workflow exists, let it handle
  builds and artifacts. The skill handles versioning, notes, and
  the GitHub release.
- **Version files are conditional.** Only update files that actually
  track the tag version. If the project derives its version from
  Git tags at build time, leave version files alone.
- **Always confirm before shipping.** Tagging and pushing are
  irreversible. The user must explicitly approve the release plan.
- **Skip generated code.** Don't run checks on packages that are
  entirely generated. They aren't real problems.
- **Start from a clean tree.** If there are uncommitted changes,
  stop before doing anything. Releases should never include
  accidental changes.
- **Release notes should be clean.** Strip redundant prefixes,
  capitalize properly, and group by type. But never rewrite the
  substance of a commit message.
- **Discover all packages.** In multi-package repos, every real
  package gets checked. A failure in any blocks the release.
