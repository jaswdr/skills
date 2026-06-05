---
name: kasa-stacked-mr
description: >
  Use when the user mentions "kasa", "stacked MR", "stacked merge requests",
  "split work into MRs", "plan commits", "atomic commits", "split into commits",
  or wants to manage a stack of merge requests on GitLab.
  Also trigger when asked to "use kasa", "create a stacked MR", "push my stack",
  "check stack status", or break work into small reviewable commits for GitLab.
allowed-tools: [Bash, Read, Glob, Grep, Edit, Write]
---

# Kasa — Stacked Merge Requests for GitLab

Kasa turns each commit on your feature branch into its own GitLab MR. MRs are chained so each diff shows only one commit's changes. State is tracked via `kasa-commit-id` trailers in commit messages — no external files.

## Mandatory: Plan Commits First

**STOP. Before writing or editing ANY code, you MUST propose a commit plan to the user and get approval.**

This is not optional. Every task that involves code changes must start with a commit plan. The plan tells the user (and future reviewers) what each MR will contain and why it is separated.

**Single-concern changes** (one cohesive feature, fix, or refactor — regardless of line count): a one-line plan is fine.
Example: *"Single commit: add role-based access service with tests"*

**Multi-concern changes** (genuinely independent pieces that can be reviewed/shipped separately): present a numbered plan:

```
Commit plan:
  1. "Refactor auth module to extract token validation" — prep for OAuth (~80 lines)
  2. "Add OAuth2 provider with tests" — new feature + tests (~250 lines)
  3. "Add OAuth login endpoint with tests" — HTTP layer + tests (~150 lines)
```

Each commit in the plan must be independently shippable — it compiles, passes CI, and makes sense without the commits above it.

**If you are already mid-implementation when this skill loads**, pause and plan the remaining work before continuing.

## The Shippable Commit Rule

**Every commit must be independently shippable.** It must compile, pass CI, and be safe to merge on its own. This is the single most important rule — it overrides all other guidance below.

A commit is a **reviewable unit**: a reviewer should be able to look at one MR, understand what it does and why, and approve it without needing to see the rest of the stack.

## Commit Splitting Rules

Split when pieces are **genuinely independent** — they can be reviewed, tested, and merged separately. Don't split just because kasa is active or because "atomic commits" sounds good.

- **Tests ship with their implementation.** A service and its tests are ONE commit. An MR without tests is unshippable. Tests without the code they test don't compile. Never separate them.
- **A feature is one commit** unless it has clearly independent parts. A model + service + controller + tests for one feature = one commit if it's one reviewable concern.
- **Separate refactors from features.** If you need to rename or restructure existing code to enable your feature, that refactor is its own commit, placed before the feature.
- **DB migrations in their own commit.** Never bundle a migration with the code that uses the new schema.
- **Config/dependency changes in their own commit** when they're not trivially part of a feature (e.g., a major dependency upgrade deserves its own MR).
- **~300 line soft limit** is a signal to *consider* splitting, not a mandate. A 400-line cohesive feature with tests is better as one commit than two broken halves.

## When NOT to Split

These should always be a single commit:
- A function/service and its tests
- A small feature (< ~200 lines) touching a few files
- Changes that would be broken, untestable, or confusing if deployed separately
- Bug fixes (the fix + the test that proves it's fixed)

## When to Split

These are good reasons to use multiple commits:
- **Independent features**: 3 unrelated API endpoints → 3 commits (each with its own tests)
- **Prep + feature**: refactor existing code, then build on it
- **Migration + code**: schema change in one commit, code using the new schema in the next
- **Large scope**: a 600-line feature with a clear seam (e.g., backend service + frontend UI)

## Commit Ordering

- **Dependencies first**: if commit B depends on commit A, A comes first.
- **Refactors before features**: prep work lands before the feature that needs it.
- **Infrastructure before consumers**: migrations, config, types — then the code that uses them.

## Stack vs. New Branch

Not all work belongs on the current stack. Before adding a commit, ask: **does this change depend on the current stack?**

- **Same stack**: the new work builds on or relates to commits already in the stack, and they'll be reviewed as a group.
- **New branch**: unrelated bug found while working, a separate feature request, or work that could merge independently. Switch to main, create a new branch, do the work there, then come back.
- **Rule of thumb**: if you could merge the new work before or after the current stack without conflicts or confusion, it should be a separate branch.

## Review-Friendly Commit Messages

Commit messages become MR titles and descriptions. Write them for reviewers:

- **Subject line** = MR title. Clear, descriptive, imperative mood ("Add auth middleware", not "Added some auth stuff").
- **Body** = MR description. Include:
  - **Why**: What problem this solves or what it enables.
  - **Stack context**: "Commit 3/5. Previous: added the User model. Next: adds the auth middleware that uses this service."
  - **Review hints**: If there is a specific part of the diff the reviewer should focus on, mention it.
  - **Breaking changes**: If this commit changes an API, renames a public function, or alters behavior, flag it.

## Mid-Work Splitting

If you realize during implementation that a commit is growing beyond its planned scope:

1. **Stop** writing new code.
2. **Identify** the natural split point in what you have written so far.
3. **Stage and commit** the complete, self-contained part.
4. **Inform the user**: "This commit was getting large. Committed [X], now continuing with [Y]."
5. **Continue** with the next piece.

If the split is not clean (half-written function), stash the incomplete work, commit the complete part, then unstash and continue.

## Commands

| Command | What it does |
|---------|-------------|
| `git kasa update` | Syncs commit stack with GitLab MRs. Creates, updates, retargets, and cleans up MRs. Use `--dry-run` to preview. First run adds trailers via rebase (needs clean working tree). |
| `git kasa status` | Shows stack overview: MR links, pipeline status, approvals. |
| `git kasa amend [target]` | Amend staged changes into a stack commit. `target` is a 1-based stack index (from `git kasa status`) or a commit hash prefix. Omit `target` for interactive picker (humans only). Use `-u` to auto-update after. |
| `git kasa absorb` | Auto-distributes staged changes to the correct commits via `git blame`. Use `-u` to auto-update after. **Preferred over amend when changes touch multiple commits.** |
| `git kasa merge` | Merges ready MRs bottom-up. Stops at first non-ready MR. Cherry-picks remaining commits onto updated default branch. |
| `git kasa clean` | Closes all kasa MRs and deletes their remote branches. |
| `git kasa restack` | Rebases the commit stack onto the latest `origin/<defaultBranch>`. Guided conflict resolution. |
| `git kasa continue` | Continue a paused restack after resolving conflicts. |
| `git kasa abort` | Abort a restack and restore the branch to its pre-restack state. |
| `git kasa fold [from] [to]` | Squash a contiguous range of commits into one. `from`/`to` are stack indices or hash prefixes (auto-swapped if reversed). Omit both for interactive picker (humans only). Use `-u` to auto-update, `-m "msg"` to set message, `--keep-first` to keep the oldest commit's message. |

## Workflow

### 1. Plan commits

See **Mandatory: Plan Commits First** above. Do not proceed to step 2 until the commit plan is approved.

### 2. Implement on a feature branch

```bash
git checkout -b <descriptive-branch-name>

# For each planned commit:
# 1. Make the focused changes
# 2. Stage only relevant files
git add <specific-files>
# 3. Commit with a clear message
git commit -m "Add User type and auth token schema" -m "Defines the User interface and JWT token schema needed by the auth middleware."
```

### 3. Sync with GitLab

```bash
git kasa update
```

This creates one MR per commit, chained so each targets the previous one's branch. First run adds `kasa-commit-id` trailers via rebase.

### 4. Check status

```bash
git kasa status
```

### 5. Amend a commit

**Prefer absorb** when staged changes touch lines from multiple commits — it splits hunks across the right commits via `git blame`:
```bash
git add <files>
git kasa absorb -u    # auto-attributes hunks and updates MRs
```

**For a single targeted commit, use `git kasa amend <target>`:**
```bash
# Run git kasa status to see the stack with 1-based indices
git kasa status

git add <files>
git kasa amend 2 -u           # amend commit #2 in the stack and sync
# or
git kasa amend abc1234 -u     # amend by hash prefix
```

The `target` is required when Claude (or any script) calls amend — omitting it triggers an interactive picker which a non-tty agent cannot drive.

If a rebase conflict occurs, resolve it and run `git kasa continue` (or `git kasa abort` to cancel).

### 6. Fold commits

Squash a contiguous range of commits into one:

```bash
git kasa fold 1 3 -u                     # fold commits 1 through 3, then sync
git kasa fold 2 3 -m "Combined message"  # custom message
git kasa fold aaa1234 ccc5678            # by hash prefix
```

By default the folded commit keeps the *last* commit's message. Use `--keep-first` for the oldest, or `-m` to override entirely. Both `from` and `to` are required for non-interactive use.

### 7. Merge when ready

```bash
git kasa merge
```

Merges the bottom MR if ready (CI passed, approved, no conflicts). Repeat until the stack is empty.

### 8. Abandon if needed

```bash
git kasa clean
```

## Important constraints

- **Clean working tree required** before `git kasa update` (first run does a rebase to add trailers). Commit or stash first.
- **Auth**: Needs `GITLAB_TOKEN` env var or `glab auth login`. If a kasa command fails with 401/403 (authentication failed), automatically attempt recovery by running `glab auth status` via Bash to refresh the token, then retry the failed kasa command. If the retry still fails, ask the user to run `! glab auth login` to re-authenticate interactively.
- **First update rewrites history**: The trailer rebase changes commit hashes. This is expected and only happens once per commit.
- **Branch naming**: `kasa/<username>/<base-branch>/<feature-branch>/<commit-id>` — don't manually create branches with this pattern.
- **MR descriptions**: Kasa injects a stack table between `<!-- kasa:start -->` and `<!-- kasa:end -->` markers. User content outside these markers is preserved.
