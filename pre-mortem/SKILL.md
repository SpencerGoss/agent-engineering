---
name: pre-mortem
description: Use when reviewing code for fragility before deployment, after completing a feature, before a release, or when the user says "what could break", "find fragile code", "pre-mortem", "what will go wrong", "future bugs". NOT for current bugs (use debug-session) or code style (use refactor-session).
license: MIT
compatibility: "Works across Claude Code, Codex, Gemini CLI, Copilot CLI, and any Agent Skills-compatible host. No runtime dependencies — pure prompt structure."
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

## Hard Rules

- **Future fragility, not current bugs.** Pre-mortem analyzes code that WORKS NOW but will plausibly break from a reasonable future edit. If something is already broken, stop and run a systematic diagnosis (debug-session) instead.
- **Every finding needs a complete causal chain.** "This could be a problem" is not a finding. Write the full incident in past tense with cause → effect → detection, or don't report it.
- **Hard cap of 7 incidents per analysis.** Pick the most plausible and highest-severity. Over-reporting buries the signal.
- **Coverage downgrades severity.** If a fragility is already protected by tests, it is not a P0. Lower its severity accordingly.

# Pre-Mortem Analysis

Write incident reports for bugs that **haven't happened yet**. Read working code and identify where future edits will plausibly introduce failures.

## Philosophy

This is NOT a linter. NOT a code review. This is a **predictive failure analysis** — imagining realistic future scenarios where reasonable edits break working code. The output is written in past tense ("What happened", "Why it broke") to force complete causal chains instead of vague warnings.

## Step 1: Scope Selection

Ask: "Which area should I pre-mortem?" or infer from context.

Scope options:
- **File**: Single file deep-dive (best for complex modules)
- **Feature**: Cross-file analysis of a feature path (best for integration risks)
- **System**: Broad scan for top fragility hotspots (broadest; parallelize across modules if the host supports it)

Read the target code. Max 5 files for File/Feature scope. For System scope, work module by module.

## Step 2: Fragility Pattern Scan

For each code area, check these 12 fragility patterns:

| Pattern | What to Look For |
|---------|-----------------|
| **Implicit ordering** | Code that works only because steps happen in a specific order, with no enforcement |
| **Shared mutable state** | Globals, module-level dicts, class variables mutated by multiple callers |
| **Stringly-typed contracts** | String comparisons for control flow (`if status == "active"`) |
| **Baked-in data assumptions** | Hardcoded column names, assumed data shapes, magic indices |
| **Coincidental correctness** | Code that produces right answers for current data but wrong for edge cases |
| **Non-atomic operations** | Multi-step mutations that can leave state inconsistent if interrupted |
| **Invisible invariants** | Rules that must hold but aren't enforced (e.g., "X must be called before Y") |
| **Load-bearing defaults** | Default values that silently mask failures instead of surfacing them |
| **Implicit resource lifecycles** | Files, connections, locks that depend on call order for cleanup |
| **Version-coupled assumptions** | Code that breaks when a dependency updates (API version, schema change) |
| **Silent None propagation** | Functions that return None on failure where callers expect values |
| **Train/inference skew** | ML paths where training and prediction use different preprocessing |

## Step 3: Write Incident Reports

For each fragility found (max 7 per analysis), write a realistic incident report:

```markdown
### Incident: [Short title]

**Severity:** P0 / P1 / P2 / P3
**Fragility Pattern:** [from table above]
**File(s):** [paths]

**What happened:** [Past tense — describe the plausible future scenario]
A developer added multi-field validation to the form processor. Each invalid field
now generates a separate error record. The success_rate metric divided total_records
by error_count, assuming one error per row. With multiple errors per row, the
denominator inflated, dropping the reported success rate from 94% to 61%.

**Why it broke:** [Root cause chain]
The success_rate calculation in metrics.py:47 assumed a 1:1 relationship between
rows and errors. This assumption was never documented or enforced. The validation
change was reasonable and passed all existing tests.

**How it was caught:** [Realistic detection path]
An A/B test showed a 30% drop in the success metric. The on-call engineer traced it
to the validation change merged 3 days prior.

**Hardening suggestion:** [Specific, minimal fix]
Add assertion: `assert error_count <= total_records` in metrics calculation.
Or: normalize by unique row IDs, not raw error count.
```

## Step 4: Output PRE-MORTEM.md

Write all incidents to `PRE-MORTEM.md` (or `.planning/PRE-MORTEM.md` if that directory exists) with:
- Header: date, scope, files analyzed
- Incidents ranked by severity (P0 first)
- Summary: total fragilities found, top 3 actionable items

## Step 5: Route to Action

| Finding | Route To |
|---------|---------|
| P0 fragility in production code | Flag to the user immediately — potential current risk |
| P1-P2 fragilities | Add to a tracked "hardening backlog" (e.g., a TODO/backlog doc) |
| Pattern appears across 3+ files | Capture it as a systemic insight in your durable notes — it's bigger than one file |
| ML train/inference skew detected | Pair with a domain expert review before deploying the model path |

## Trigger Conditions

- Reviewing code for fragility before a deployment or release
- Just completed a feature and want to know where it will bite later
- The user says "what could break", "find fragile code", "pre-mortem", "what will go wrong", or "future bugs"
- A working module is about to take on new callers or new edits

## Out of Scope

- Current bugs or test failures → run a systematic diagnosis (debug-session)
- Code style or naming → refactor-session
- Security vulnerabilities → security-audit
- Performance bottlenecks → performance-tuning
- Quality review of code as written (not future fragility) → code-review-session

## Common Traps

- **Finding current bugs, not future fragilities** — pre-mortem is about code that WORKS NOW but will break from reasonable future edits. If it's already broken, use debug-session.
- **Vague warnings without causal chains** — "This could be a problem" is useless. Write the full incident with past-tense causality.
- **Over-reporting** — Max 7 incidents. Pick the most plausible and highest-severity. Don't report style issues.
- **Ignoring test coverage** — If a fragility is already covered by tests, it's not a P0. Downgrade severity.
