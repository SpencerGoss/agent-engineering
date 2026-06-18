---
name: spec-driven-dev
description: >
  Use when about to build or implement a specific feature with clear scope — before writing
  implementation code. Trigger on: "implement [feature]", "build [feature]", "add [feature]",
  "create a system that [does X]", "planning session for [feature]", or an explicit feature
  request with defined scope. NOT for: vague "let's work on X", simple additions, config
  changes, bug fixes, or exploratory discussion.
license: MIT
compatibility: "Works across Claude Code, Codex, Gemini CLI, Copilot CLI, and any Agent Skills-compatible host. No runtime dependencies — pure prompt structure."
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

## Hard Rules

- **No implementation code before a spec file exists.** The spec is the hard gate. Create it at `docs/specs/YYYY-MM-DD-<feature-name>.md` (create `docs/specs/` if it doesn't exist) and fill every section before touching code.
- **A spec file survives context compaction; chat does not.** Plans that live only in the conversation get lost at the next reset or tool switch. The spec file is the single source of truth.
- **Never state file contents or implementation state from memory after a context reset.** Always re-read the spec and re-run commands to get current output.
- **Every Open Question must be resolved (or explicitly marked non-blocking with rationale) before implementation starts.** Every unresolved "TBD" is a future bug.
- **Acceptance criteria must be testable** — a pass/fail assertion you can actually write. If you can't, rewrite the criterion until you can.

---

# Spec-Driven Development

Write the spec before the code. No implementation until the spec file exists.

**Why:** Plans that live in chat get lost when context is compacted. A spec file survives compaction, session resets, and tool switches. It's the single source of truth that the next session — or the next developer — can pick up from cold.

---

## Spec File Format

```markdown
# Spec: <Feature Name>
Date: YYYY-MM-DD
Status: IN PROGRESS | COMPLETE

## Goal
One sentence: what does this feature do for the user?

## Non-Goals
What this feature explicitly does NOT do (prevents scope creep).

## User Stories
- As a [user], I want [action] so that [outcome]
- As a [user], I want [action] so that [outcome]

## API Contracts (if applicable)
Input: what goes in (type, shape, constraints)
Output: what comes out (type, shape, guarantees)
Errors: what can go wrong and what is returned

## Data Models (if applicable)
Fields, types, relationships, constraints

## Acceptance Criteria
- [ ] Criterion 1 (testable: "when X happens, Y is true")
- [ ] Criterion 2
- [ ] Criterion 3

## Implementation Tasks
- [ ] Task 1
- [ ] Task 2
- [ ] Task 3

## Assumptions
- [VERIFIED] Assumption 1 — checked via: [how you verified it]
- [UNVERIFIED] Assumption 2 — needs test: [what test would prove/disprove it]

## Open Questions
- Question 1 (must be answered before starting)
```

---

## Step 0: Step Back (before requirements)

**Pick the right tool first.** Not every task belongs in a coding agent — a UI prototype, a one-off data analysis, a simple form/tracker, or a document-synthesis question may be better served elsewhere. Production code, complex logic, and multi-file changes are where this workflow earns its keep. Make that call before you start planning.

Before gathering requirements, ask:

1. **What category of work is this?** (new feature, extension, integration, migration, optimization, fix)
2. **What's the general pattern for this category?**
   - New feature: define boundaries first, then internals
   - Extension: understand existing code fully before proposing changes
   - Integration: define the contract/interface before implementation
   - Migration: write the migration plan before touching code
   - Optimization: profile before optimizing, measure before and after
   - Fix: reproduce first, understand root cause, fix minimal surface area
3. **What similar work exists in this codebase?** (grep for similar patterns)

The category determines your approach. Don't start gathering requirements until you know what kind of work this is.

**For novel architecture or unfamiliar domains:** Run 2-3 divergent research passes before committing to an approach. Exploring multiple directions produces better architecture than serial thinking down one path.

**For significant decisions:** Present options with tradeoffs before committing. Name at least one rejected alternative and why it lost — show the *why*, not just the choice.

---

## Workflow

1. **Write the spec** — fill in every section above before touching code.
2. **Resolve open questions** — don't start with unknowns.
3. **Spec Reflection** (before implementing) — critique your own spec before writing any code:
   - What requirement am I most likely misunderstanding?
   - What edge case will the user report first?
   - Is this spec solving the right problem, or the problem I assumed?
   - Step-back: What general category of solution is this? (CRUD, pipeline, UI, integration, algorithm) — am I using the right pattern for this category?
   - If any answer changes the spec, revise it now. It's 10x cheaper to fix a spec than fix code.
4. **Surface and test assumptions** — before implementation, list every implicit assumption:
   - "What am I assuming about the data?" (shape, nulls, types, ordering)
   - "What am I assuming about the API?" (availability, rate limits, response format)
   - "What am I assuming about the environment?" (packages, paths, permissions)
   - "What am I assuming about existing code?" (interfaces, side effects, thread safety)
   - Mark each: VERIFIED (checked with code/docs) or UNVERIFIED (needs a test)
   - Add an `## Assumptions` section to the spec file with this list
   - Unverified assumptions become the FIRST test cases when you write tests
5. **Hand off to the test workflow** — use acceptance criteria + unverified assumptions as your initial test cases (see **tdd-workflow**).
6. **Track progress in the spec file** — mark tasks `[x]` as done.
7. **Log the work** — cross-reference the spec file path in your project log.

---

## Resume Protocol (After Context Compaction)

When context resets or compaction runs, ALWAYS do this before making any claims:

1. Read the spec file: `docs/specs/YYYY-MM-DD-<feature>.md`
2. Check which tasks are `[x]` (done) vs `[ ]` (pending)
3. Resume from the first incomplete task
4. Never assume state from memory — always re-read the spec

**Anti-hallucination rule:** Do not state file contents or implementation state from memory. Always re-read the spec file. Always re-run commands to get current output. Memory is unreliable after compaction.

---

## Progress Tracking

Mark tasks done in the spec file as you complete them:
```markdown
## Implementation Tasks
- [x] Create database schema migration
- [x] Write repository layer
- [ ] Add API endpoint  ← current task
- [ ] Wire up frontend
```

This survives context compaction. Chat history does not.

---

## Quick Reference

| Situation | Action |
|-----------|--------|
| Tempted to code without a spec | Stop. Write the spec first. |
| Open questions exist | Answer them before writing code |
| Context was compacted | Re-read spec file before continuing |
| Feature scope is expanding | Update spec Non-Goals section, discuss with user |
| Spec acceptance criteria unclear | Rewrite until they're testable (observable pass/fail) |
| Session ended mid-feature | The spec file gives full resume context for the next session |

---

## Trigger Conditions

- "implement [feature]", "build [feature]", "add [feature]"
- "create a system that [does X]"
- "planning session for [feature]"
- An explicit feature request with defined scope, before any implementation code is written

---

## Out of Scope

- NOT for bug fixes or debugging existing code — run a systematic diagnosis instead (see **debug-session**).
- NOT for writing tests or implementing code after the spec is done — use **tdd-workflow** for RED/GREEN/REFACTOR.
- NOT for quick one-line changes that don't need a plan — specs are for non-trivial features only.
- NOT for vague, exploratory discussion with no defined scope.
- NOT for updating project documentation or context files — that's a separate concern.

---

## Common Traps

- **Spec drifts from implementation:** The spec says one thing but the code evolved differently during implementation. Always update the spec file when requirements change mid-build — a stale spec misleads the next session or developer who reads it.
- **Over-specifying internals:** Specs should describe *what* and *why*, not *how*. Specifying internal function names, variable names, or implementation patterns locks you into decisions that should be made during coding. Keep API Contracts and Acceptance Criteria external-facing.
- **Open Questions left unresolved:** Starting implementation with unresolved Open Questions leads to assumptions that become wrong. Every "TBD" in a spec is a future bug — resolve them before writing code or explicitly mark them as non-blocking with rationale.
- **Acceptance Criteria that aren't testable:** "The UI should feel fast" is not testable. "Page load completes in under 2 seconds" is. If you can't write a pass/fail assertion for a criterion, rewrite it until you can.
- **Spec as a one-time artifact:** Teams write the spec, then never look at it again. The spec should be the living checklist — mark tasks `[x]` as you go, and re-read it after context compaction. If nobody references the spec during implementation, it wasn't useful.
