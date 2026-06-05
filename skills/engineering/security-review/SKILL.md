---
name: security-review
description: >
  Security-focused review of a diff or set of files. Runs two parallel
  sub-agents — one checking the OWASP Top 10, one checking project-specific
  security standards — then aggregates findings by severity with a suggested
  fix per finding. Use when the user wants to security-review a branch, a PR,
  a file, or a directory, or says "check this for security issues", "audit
  this for vulnerabilities", or "run a security review".
argument-hint: "Branch, commit, tag, or file/directory path to review"
---

# Security Review

Two-axis parallel security review: **OWASP Top 10** (universal vulnerability classes) and **Project Standards** (repo-specific security rules). Both axes run as independent sub-agents so neither pollutes the other's context.

The issue tracker and project docs should be available — run `/setup-jaswdr-skills` if `docs/agents/issue-tracker.md` is missing.

## Process

### 1. Determine the target

Two modes:

**Diff mode** — the user provides a fixed point (branch, commit, tag, `main`, `HEAD~5`, etc.). The review covers only what changed:

```bash
git diff <fixed-point>...HEAD
git log <fixed-point>..HEAD --oneline
```

**Path mode** — the user provides a file or directory path. The review covers the current state of those files. List them:

```bash
find <path> -type f | head -60
```

If the user didn't specify, ask: *"Review a diff against a fixed point, or specific files/directories?"* Don't proceed until you have it.

### 2. Collect project-specific security standards

Read every file that documents security rules for this repo. Common locations:

- `SECURITY.md`, `docs/security.md`, `docs/SECURITY.md`
- `CLAUDE.md`, `AGENTS.md` (look for security-relevant rules)
- `CONTEXT.md` (authentication, authorisation, data handling decisions)
- `docs/adr/` (ADRs that encode security decisions)
- `.env.example` (reveals what secrets the app uses — useful context)

If none exist, note "no project-specific security standards found" and rely on OWASP alone.

### 3. Spawn both sub-agents in parallel

Send a single message with two `Agent` tool calls using `general-purpose` for both.

---

**OWASP sub-agent prompt** — include:

- The diff command or file list from step 1.
- The current OWASP Top 10 categories:
  - A01 Broken Access Control
  - A02 Cryptographic Failures
  - A03 Injection
  - A04 Insecure Design
  - A05 Security Misconfiguration
  - A06 Vulnerable and Outdated Components
  - A07 Identification and Authentication Failures
  - A08 Software and Data Integrity Failures
  - A09 Security Logging and Monitoring Failures
  - A10 Server-Side Request Forgery (SSRF)
- The brief: *"Read the diff/files. Report every security finding mapped to an OWASP Top 10 category. For each finding: severity (Critical/High/Medium/Low), OWASP category, file + line, description of the vulnerability, and a concise suggested fix (1–3 lines). If nothing is found for a category, omit it. Under 600 words total."*

---

**Project Standards sub-agent prompt** — include:

- The diff command or file list from step 1.
- The contents or paths of every project-specific security file found in step 2.
- The brief: *"Read the project security standards. Then read the diff/files. Report every place the code violates a documented security rule. For each finding: severity (Critical/High/Medium/Low), the rule violated (cite the source file + rule), file + line, description, and a concise suggested fix (1–3 lines). If no violations are found, say so explicitly. Under 400 words."*

---

### 4. Aggregate findings

Merge the two reports into a single output grouped by severity:

```
## Critical
…

## High
…

## Medium
…

## Low
…

## Summary
X findings total (Y Critical, Z High, …). Worst: <one-line description of the most severe issue>.
```

Format each finding as:

```
**[SEVERITY] <short title>** `file:line`
OWASP: <category> | Source: <OWASP / project-standards file>
<1–2 sentence description of the vulnerability.>
Fix: <1–3 lines describing the remediation.>
```

De-duplicate if both sub-agents found the same issue — keep the more detailed description and note both sources.

If one sub-agent found nothing, include a line: *"No OWASP / project-standards violations found."*

## Why two axes

A change can pass one and fail the other:

- Code that follows project security rules but introduces a classic injection vector → **Project Standards pass, OWASP fail.**
- Code that avoids all OWASP categories but stores a secret in a way the team has explicitly forbidden → **OWASP pass, Project Standards fail.**

Reporting them separately ensures neither masks the other, and the source of each finding is always clear.
