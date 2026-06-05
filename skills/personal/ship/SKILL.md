---
name: ship
description: End-to-end ship workflow — first asks whether to use a git worktree or a simple branch, then creates a Linear issue assigned to the current user, syncs master, creates a `feature/<issue-id>` (or `fix/<issue-id>` for bugs) branch (or worktree), decomposes the work into a dependency-aware todo plan and executes independent todos in parallel via subagents, then commits and pushes, opens a GitLab MR assigned to the current user, and prints a summary table with Linear issue / branch / commit / MR. Use when the user invokes `/ship` or says "ship this", "ship feature X", "do the full flow", or otherwise asks for the end-to-end Linear → branch → code → commit → push → MR pipeline.
---

# /ship — end-to-end ship workflow

Run the full pipeline below in order. Do NOT collapse or skip steps. If a step fails, stop and report — do not paper over failures.

## 0. Ask: worktree or simple branch? (FIRST — before anything else)

Before parsing the request, before resolving identities, before any tool call other than this one, ASK the user with `AskUserQuestion` whether they want to:

- **Worktree** — create a new git worktree (parallel checkout in a sibling directory). Use this when the user wants to keep their current checkout untouched, run multiple branches side-by-side, or work in isolation. The full ship pipeline will run inside the new worktree directory.
- **Simple branch** — create a regular branch in the current checkout (`git checkout -b ...`). This is the classic flow.

Capture the answer as `branch_mode` (`worktree` or `simple`) and use it in step 3. Do NOT proceed to step 1 until the user has answered — this choice changes where the rest of the pipeline runs.

If the user has already explicitly indicated their preference in the original `/ship` prompt (e.g. "ship this in a worktree", "just a branch is fine"), honor that and skip the question.

## 1. Parse the request

Extract from the user's prompt:

- **Linear team** (e.g. ENG, PLATFORM, INFRA). If not stated, ASK before doing anything else — do not guess. Use `AskUserQuestion`.
- **Branch kind**: default `feature/`. Use `fix/` if the user calls this a bug, says "fix", "bug", "regression", "broken", or the work is clearly remedial.
- **Issue title**: derive a concise, accurate title from the requested work. Imperative voice, present tense, no trailing period.
- **The work itself**: everything else — the actual change to make.

Do NOT ask the user to confirm the title or description — proceed. The user can edit Linear/MR after if needed.

## 1a. Resolve the current user

You need two identities for this run:

- **Linear assignee name** (full name as it appears in Linear, e.g. `John Doe`).
- **GitLab handle** (username without the `@`, e.g. `johndoe`).

Resolve them in this order:

1. If the user explicitly named someone for either role in this prompt, use that.
2. Try to discover them automatically:
   - GitLab handle: `glab auth status` (look for the logged-in username) or `glab api user --jq .username`.
   - Linear name: query the Linear MCP for the current viewer/me. If the MCP exposes no "me" call, fall back to asking.
3. If either is still unknown, ASK with `AskUserQuestion` before proceeding. Do not guess and do not default to a hardcoded name.

Cache both values for the rest of the run.

## 2. Create the Linear issue (FIRST — needed for branch name)

Use the Linear MCP `save_issue`:

- `team`: from step 1
- `assignee`: the Linear name resolved in step 1a
- `title`: from step 1
- `priority`: `2` (High) for bugs, `3` (Normal) otherwise — unless the user specified
- `description`: a real description, not a placeholder. Include:
  - Summary of the problem or goal
  - File paths and line numbers where the change happens (if known)
  - For bugs: reproduction steps + expected vs actual
  - For features: scope + acceptance criteria
  - Use `\n` literal newlines in the markdown — do NOT escape

Capture the returned `id` (e.g. `ENG-2844`) and `url`. Lowercase the ID for the branch name (`eng-2844`).

## 3. Sync master and create the branch

Use the `branch_mode` captured in step 0.

### 3a. Simple branch (`branch_mode == simple`)

Run from the **repo root**:

```bash
git checkout master
git pull --ff-only
git checkout -b <feature|fix>/<lowercase-issue-id>
```

If `git pull` is non-fast-forward, stop and ask — do not force.

If the working tree is dirty, stop and ask — do not stash without permission.

### 3b. Worktree (`branch_mode == worktree`)

From the repo root, create the worktree and branch together:

```bash
git worktree add ../<repo-name>-worktrees/<branch-name> -b <feature|fix>/<lowercase-issue-id> master
```

If the repo provides a custom worktree script (e.g. `./devtools/make_worktree.sh`), prefer that over plain `git worktree add`.

After the worktree is created, **`cd` into it** and run every remaining step (4 onward) from inside the worktree. Do not return to the original checkout for commits, pushes, or MR creation — the new branch only exists in the worktree.

## 4. Analyze the problem and build a dependency-aware todo plan

Before writing any code, decompose the requested work into discrete todos and figure out which can run in parallel. The goal is to maximize parallelism without two agents ever fighting over the same file — that's the failure mode this step exists to prevent.

Work through it like this:

1. **Decompose.** Break the request into the smallest todos that each produce a coherent, independently-implementable change (e.g. "add the schema migration", "add the changeset + validations", "wire the handler", "add the LiveView", "write the integration test"). Aim for todos a single focused subagent can finish on its own.

2. **Assign each todo a file set.** For every todo, estimate the concrete files it will create or modify (paths). This is the heart of the strategy: parallelism is decided by file overlap, not by intuition. Two todos may run in parallel **only if their file sets are disjoint**. If you can't confidently predict a todo's files, treat it as overlapping with everything (i.e. don't parallelize it) — guessing wrong causes lost work.

3. **Map blockers.** A todo is *blocked by* another when it needs the other's output to exist first — it edits a module/function the other creates, imports a type the other defines, or tests behavior the other implements. Record each todo's `blocked_by` list. A blocked todo can never share a wave with the todo it depends on.

4. **Group into waves.** Wave 1 = every todo with no blockers *and* a file set disjoint from the others in the wave. If two otherwise-ready todos share a file, keep one in wave 1 and push the other to a later wave (or merge them into one todo if splitting them is artificial). Each subsequent wave = todos whose blockers all completed in earlier waves, again filtered so the wave's file sets are mutually disjoint. Most tasks need 1–3 waves.

5. **Record the plan in the todo list.** Use `TodoWrite` so the user sees live status. Write each todo so its in-progress state reads clearly, and note its wave + `blocked_by` so the structure is visible.

### Approval checkpoint (conditional)

- **Small/simple plans** — **2 or fewer todos and no cross-todo dependencies**: skip approval, go straight to step 5. Adding ceremony to a one-file change wastes everyone's time.
- **Bigger or dependent plans** — more than 2 todos, or any `blocked_by` relationship: present the plan and wait for an OK before spawning anything. Render it as a table so the dependency structure is legible, then ask with `AskUserQuestion` (options: **Looks good, proceed** / **Let me adjust**):

  ```
  | # | Todo | Files | Blocked by | Wave |
  |---|---|---|---|---|
  | 1 | Add DB migration | db/migrations/..._x.sql | — | 1 |
  | 2 | Add model + validations | src/models/x.ts | — | 1 |
  | 3 | Wire handler | src/handlers/x.ts | 2 | 2 |
  ```

  If the user wants changes, revise the decomposition and re-present. Don't spawn subagents until the plan is accepted.

If the whole task is genuinely a single small change, this step collapses to a one-line plan — that's fine. Don't manufacture todos to justify parallelism.

## 5. Execute the plan (parallel waves)

Run the waves in order. Within a wave, spawn **one subagent per todo, all in the same turn**, so they execute in parallel. Then barrier-wait for the entire wave to finish before starting the next wave — dependent todos need their blockers' files to already exist in the working tree.

**Picking the subagent type:** Use `general-purpose` by default. Match a project-specific subagent type if available.

**Brief each subagent with:**

- The specific todo and **its assigned file set**, with an explicit constraint: *edit only these files. If you discover you need to touch a file owned by another todo, STOP and report it instead of editing — do not modify files outside your set.* This is what keeps parallel agents from clobbering each other.
- The Linear issue context (problem + acceptance criteria) so it implements the right thing.
- The code conventions below (what to write — not what to run).
- An explicit boundary: **edit source files only. Do NOT run the build tool, do NOT run git.** Parallel agents share one working tree, so concurrent builds race and produce spurious failures — the orchestrator owns all building and git. Return a short summary of what changed (files + what each does).

**Verify after each wave (orchestrator, not subagents).** Once a wave's agents all return, run the build/format/tests yourself — serially, in the one checkout, where there's no race:

- Run the project's formatter on the wave's changed files.
- Run focused tests on the touched files only. NEVER a project-wide test run when a focused one is possible.
- If the build or tests fail, fix it (or send a targeted follow-up to the relevant agent) before launching the next wave — dependent todos build on this wave's code, so a broken wave poisons everything downstream.

**Between waves:** the edited files are already in the shared working tree (same checkout). Mark completed todos done in `TodoWrite`, then launch the next wave. If a subagent stopped because it hit a foreign file (a file-set collision the plan missed), serialize that todo into a later wave instead of re-running it in parallel.

**For a trivial single-todo plan**, skip the subagent machinery and just do the work inline — fanning out one agent adds latency for nothing.

**Code conventions every subagent (and any inline work) must follow:**

- New source files require a corresponding test file.
- Keep changes scoped — don't sweep up unrelated edits.
- Building/formatting/testing is the orchestrator's job — subagents edit files only.

**After the final wave**, do one full pass: format and run the focused tests across the complete set of touched files before moving to review.

## 5a. Code review (before commit)

Spawn a code-review subagent to review the changes you just made. Use the `Agent` tool with a `code-reviewer` subagent type if available, otherwise `general-purpose`.

Brief the agent with:

- The list of changed file paths (from `git status` / `git diff --name-only master...HEAD`).
- The Linear issue context (problem + acceptance criteria) so it can judge intent vs. implementation.
- An explicit ask: **categorize every finding by severity — `Critical`, `P1`, `P2`, or `Nit` — and report each with file path + line number**.

Wait for the report and capture the findings list verbatim. Do NOT collapse findings into your own summary — you will hand them to the fix agent in step 5b.

## 5b. Address Critical and P1 findings

Filter the reviewer's findings:

- **Critical** and **P1** → must be fixed in this MR. Continue with this step.
- **P2** and **Nit** → do NOT auto-fix. List them in the MR description (step 8) as known follow-ups.

If there are no Critical or P1 findings, skip the rest of this step.

Otherwise spawn a second subagent to apply the fixes. Use `Agent` with `subagent_type: plan-executor` (or `general-purpose` if plan-executor is unavailable). Brief it with:

- The exact Critical + P1 findings copied verbatim from step 5a (severity, file:line, description).
- An explicit instruction: address these findings and **only** these; do not refactor surrounding code; do not touch P2/Nit findings; do not introduce unrelated changes.
- Project conventions (CLAUDE.md / AGENTS.md) the agent must follow — same rules as step 5.

After the fix agent finishes:

- Re-run focused tests on the touched files.
- Run the project's formatter on changed files.
- Verify with `git diff` that the changes match the findings — nothing extra crept in.

If any Critical/P1 finding cannot be addressed (e.g. requires out-of-scope work), STOP and report to the user before continuing — do not silently skip.

## 5c. User review checkpoint (before commit)

Before doing ANYTHING in step 6 (commit) or beyond, STOP and ask the user with `AskUserQuestion` whether they have reviewed the changes and want to continue.

Use exactly two options:

- **Yes, continue** — proceed to commit, push, and open the MR.
- **I have changes** — the user wants to request edits. They can describe the changes either by selecting this option and elaborating, or by typing into the "Other" free-text field.

Suggested phrasing: *"Have you reviewed the changes? Ready to commit and push, or do you want to make changes first?"*

Handling the answer:

- If **Yes, continue** → proceed to step 6.
- If **I have changes** (or any custom "Other" answer describing edits) → treat the user's note as the new instruction. Loop back to step 5 to apply the requested changes (re-plan in step 4 first if the request is large enough to warrant new parallel todos). After applying, re-run step 5a/5b only if the change is non-trivial enough to warrant another review pass; otherwise come straight back here and ask again. Do NOT proceed to step 6 until the user picks **Yes, continue**.

Do not commit, push, or open the MR before this confirmation. This checkpoint is mandatory even if the code-review subagent reported zero findings.

## 6. Commit

- Conventional commit format: `<type>(<scope>): <subject> (<ISSUE-ID>)`
  - `type`: `fix` for bugs, `feat` for features, `chore`/`refactor`/`docs` as appropriate.
  - `scope`: the closest domain (e.g. `auth`, `payments`, `api`).
- Body: explain WHY, not what. Reference the Linear ID at the end as `Closes <ID>.`
- Stage explicit paths (no `git add -A` / `git add .`).
- Do NOT add Claude/AI co-author signatures.
- NEVER `--no-verify` unless the user explicitly asks.

If a pre-commit hook fails: fix the underlying issue and create a NEW commit (do not `--amend`).

## 7. Push

```bash
git push -u origin <branch>
```

## 8. Open the MR

Use `glab` directly (or a `gitlab-access` skill if available):

- `--target-branch master`
- `--source-branch <branch>`
- `--assignee <gitlab-handle>`  ← the GitLab handle resolved in step 1a (without the `@`); use whoever the user named instead if they specified
- `--title` matches the commit subject
- `--description` filled from the project's MR template (`.gitlab/merge_request_templates/default.md`). Replace template placeholders with real content. Include:
  - Problem summary
  - Before/after code blocks if the diff is small enough to be illustrative
  - Link to the Linear issue
  - Tick the correct "Type of Change" box (Bug fix vs New feature)
  - Tick "Added tests" only if you actually added tests; reference them by name
- Use `--yes` to skip interactive prompts.
- Use a HEREDOC or temp file for the description so newlines render literally.
- Do NOT mention Claude/AI in the description.

## 9. Print the summary table

After everything is created, print exactly this table to the user (Markdown, three columns is fine — adjust if values are long):

```
| Step | Result |
|---|---|
| **Linear** | [<ID>](<url>) — assigned to <linear-name> |
| **Branch** | `<branch>` |
| **Commit** | `<short-sha>` — `<commit-subject>` |
| **MR** | <mr-url> — assigned to `@<gitlab-handle>` |
```

Use the actual short SHA from `git rev-parse --short HEAD` after committing.

## Failure handling

- If any step fails: STOP. Report what failed and what was already created (Linear issue may exist even if branch didn't). Do not roll back the Linear issue automatically — ask first.
- If the user interrupts or rejects mid-flow: leave state as-is and report the partial summary.

## Defaults reference

| Field | Default | Override when |
|---|---|---|
| Branch mode | Ask in step 0 — no default | User explicitly requested worktree/simple in prompt |
| Linear assignee | Current user (resolved in step 1a) | User names someone else |
| Branch prefix | `feature/` | Bug-shaped work → `fix/` |
| Priority | Normal (3) | Bug → High (2); user specifies |
| MR assignee | Current user's GitLab handle (resolved in step 1a) | User explicitly names someone else |
| MR target | `master` | User specifies |
| Co-author signature | None | Never |
