---
name: spike
description: >
  Scope-boxed technical investigation to validate a hypothesis before
  committing to an approach. Interviews the user to narrow the question,
  proposes a scope for approval, then reads the codebase, searches the web
  if needed, and executes in small validated steps — reflecting on direction
  after each. Produces a spike doc at `.scratch/spikes/<slug>.md` and
  ephemeral prototype code. Use when the user says "spike this", "investigate
  whether X", "I don't know if approach Y will work", or wants to validate a
  technical hypothesis before building for real.
argument-hint: "What hypothesis do you want to validate?"
---

# Spike

A spike is a scope-boxed investigation to answer a specific technical question. The output is a decision: **validated** (proceed with confidence) or **invalidated** (try something else). Code produced is ephemeral. The spike doc is the real deliverable.

## Phase 1 — Narrow the question

Interview the user to sharpen the hypothesis before doing any investigation. Ask one question at a time. For each, give your recommended answer.

Drive toward answers to:

- **What exactly is being validated?** Push from vague ("should we use X?") to a testable claim ("X can handle N concurrent writes without queue backlog").
- **What does success look like?** A benchmark? A working prototype? A yes/no from the docs?
- **What does failure look like?** What result would make you choose a different approach?
- **What is explicitly out of scope?** What won't be investigated, even if interesting?
- **What is already known?** Prior attempts, half-read docs, gut feelings — capture them as the Hypothesis.

Stop when you can write a crisp one-sentence hypothesis: *"We believe X is true, and we'll know we're right when Y."*

## Phase 2 — Propose scope and get approval

Create the spike doc at `.scratch/spikes/<slug>.md` (slug = kebab-case summary of the question). Write the first three sections and stop:

```markdown
# Spike: <question>

## Hypothesis
<One sentence: "We believe X, and we'll know we're right when Y.">

## Approach
<Bullet list of exactly what will be investigated — the fixed surface of this spike.
Each bullet is a concrete action: read a file, run a query, write a minimal prototype,
check a benchmark. Nothing vague.>

## Out of scope
<What this spike will not cover, even if tempting.>
```

Present the Approach to the user and ask for approval before proceeding. If they want changes, update the doc and re-present. Do not start Phase 3 until approved.

## Phase 3 — Investigate

Work through the Approach bullet by bullet. After each bullet, append findings to the `## Progress` section of the spike doc.

### Step order

1. **Read the codebase** — understand the current state: relevant files, existing abstractions, constraints, prior attempts. Don't assume; read.
2. **Search the web** — only if the question requires external knowledge (library docs, benchmarks, prior art). Skip if the answer is entirely in the codebase.
3. **Build the plan** — based on what you've found, write a concrete implementation plan: ordered steps, each small and independently testable.
4. **Execute in small steps** — implement one step at a time. All prototype code goes in `.scratch/`. After each step:
   - Run the relevant tests or validation (build, lint, a focused test).
   - If it passes, note it in `## Progress` and continue.
   - If it fails, diagnose before moving on — do not accumulate broken steps.

### After each completed step

Pause and reflect in `## Progress`:
- Does the result still support the hypothesis?
- Is the approach still correct, or has something changed?
- Is there a cleaner path to the answer?

If the approach needs to change, update `## Approach` and note the revision. Do not silently deviate.

## Phase 4 — Conclude

Once the Approach is exhausted (or the hypothesis is clearly validated/invalidated early), write the final two sections:

```markdown
## Prototype
<Links or inline snippets of the key prototype code in `.scratch/`. One paragraph
explaining what the prototype does and how to run it, if applicable.>

## Conclusion
<Two parts:>
<1. Verdict: "Validated" or "Invalidated" — one sentence with the evidence.>
<2. Recommendation: what to do next. If validated: proceed with X approach. If
invalidated: the alternative to consider. Keep it to 3–5 sentences.>
```

Present the Conclusion to the user. The spike is done.

## Rules

- **Prototype code is ephemeral.** Everything in `.scratch/` is throwaway. Do not refactor it, do not wire it into production paths, do not commit it to the main codebase.
- **The doc is the deliverable.** A spike with a working prototype but no Conclusion failed. A spike with a clear Conclusion but no prototype succeeded.
- **Scope discipline.** If you discover something interesting outside the Approach, note it in a `## Tangents` section at the bottom — do not chase it. The scope is fixed.
- **Fail fast.** If the hypothesis is clearly invalidated in step 2, stop and conclude. Don't run the full Approach out of process.
