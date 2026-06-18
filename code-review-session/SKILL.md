---
name: code-review-session
description: >
  Use after writing or modifying code, before committing or merging — to catch bugs, security
  issues, and quality problems. Trigger on: "review this code", "check my code", "look at what I
  wrote", "code review", finishing a feature or bug fix, after a tdd-workflow green phase, or
  before any git commit on non-trivial changes. NOT for design reviews, architecture discussions,
  or general issue checking — only for reviewing code changes (git diff).
license: MIT
compatibility: "Works across Claude Code, Codex, Gemini CLI, Copilot CLI, and any Agent Skills-compatible host. No runtime dependencies — pure prompt structure."
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

## Hard Rules

- **Scope is the diff only** — review only what changed (`git diff` or `git diff --staged`). Never audit the entire codebase; that is a full security audit, not a code review.
- **Diff >1000 lines**: STOP and ask the user to break it into logical chunks before reviewing.
- Do not request architecture changes in a code review — that belongs to a design/architecture pass.
- Do not request refactoring for code NOT in the diff — stay in scope.
- Do not flag style issues if there is a linter/formatter configured — trust the tooling.

# Code Review Session

Review changed code systematically. Produce a severity-ranked finding list. Ship confidently.

**Diff Size Check:**
If the diff is >1000 lines, STOP and ask the user to break the review into logical chunks.
Large diffs cause reviewers to miss critical issues — smaller reviews are higher quality.

**Scope:** Review only what changed — `git diff` or `git diff --staged`. Never audit the entire codebase; that is a full security audit.

---

## Review Checklist (in priority order)

Work top to bottom. Stop and fix before moving down.

### CRITICAL — Block commit
- [ ] **Secrets exposed:** API keys, passwords, tokens hardcoded in source code
- [ ] **Data loss:** destructive operations (DELETE, DROP, overwrite) without guard
- [ ] **Provably wrong logic:** off-by-one, inverted condition, wrong operator
- [ ] **Security hole:** unvalidated user input at a boundary, SQL injection, XSS
- [ ] **New code has zero test coverage** (unless it's configuration/markup only)

### HIGH — Fix before merging
- [ ] **Exceptions swallowed:** `except: pass`, `.catch(() => {})`, silent error discard
- [ ] **Null/undefined unguarded:** accessing `.property` on something that could be None/null
- [ ] **Breaking change:** function signature changed, callers not updated
- [ ] **Performance trap:** O(n²) in hot path, N+1 queries, blocking the event loop
- [ ] **Zero test coverage on new code:** If diff introduces new functions/classes with no corresponding tests, flag as HIGH
- [ ] **Test tests the mock, not the code:** If new tests mock so thoroughly that removing the implementation would still pass, flag as HIGH
- [ ] **Coverage regression:** If coverage report shows new code is <50% covered, flag as HIGH

### MEDIUM — Fix if quick, log if not
- [ ] **Misleading names:** function or variable name doesn't describe what it does
- [ ] **Duplicated logic:** same code copy-pasted in 2+ places (extract it)
- [ ] **Magic values:** hardcoded numbers or strings that should be named constants
- [ ] **Function too long:** over 50 lines — candidate for splitting

### LOW — Note and move on
- [ ] Missing comment on non-obvious logic
- [ ] Formatting inconsistency (prefer linter over manual fix)
- [ ] TODO left in code that should be a tracked ticket

---

## How to Run

1. **Get the diff:**
```bash
git diff          # unstaged changes
git diff --staged # staged changes
git diff HEAD~1   # last commit
```

2. **Go through checklist top-down.** Don't spend time on LOW if CRITICAL exists.

3. **File each finding** with this format:
```
[SEVERITY] path/to/file.py:42 — what's wrong → specific fix
```

4. **Adversarial Pass (Devil's Advocate):**
After the checklist, flip your mindset. You are now an attacker trying to BREAK this code.
- "What input would crash this?" — pick 3 edge cases and mentally trace them
- "What assumption is wrong?" — list every implicit assumption, challenge each one
- "What happens under load/concurrency?" — race conditions, shared state, non-atomic ops
- "If I deleted the implementation but kept the tests, would tests still pass?" — mock integrity
- 60%+ of effort here should be DISCONFIRMATION, not confirmation. Ask "why X fails" not "why X works"
- If this pass finds something the checklist missed, promote it to the appropriate severity level

5. **Output a verdict:**
- ✅ **Ship** — no CRITICAL or HIGH findings
- ⚠️ **Fix first** — HIGH findings present
- 🚫 **Block** — CRITICAL found

---

## Output Format

```
Code Review — <feature or file name>
Reviewed: <git diff range>

CRITICAL (N)
- [CRITICAL] src/auth.py:15 — API key hardcoded → move to os.environ.get("API_KEY")

HIGH (N)
- [HIGH] src/api.js:88 — fetch() not awaited → add await

MEDIUM (N)
- [MEDIUM] src/utils.py:30 — magic number 86400 → extract as SECONDS_PER_DAY

LOW (N)
- [LOW] src/models.py:55 — missing docstring on public method

Verdict: ✅ Ship / ⚠️ Fix HIGH before merge / 🚫 Block
```

---

## What NOT to Flag

- **Style/formatting** a linter handles → run the linter instead
- **"I would have done it differently"** preferences → only flag if correctness is affected
- **Architectural concerns** already decided → raise separately, don't block this review
- **Test edge cases beyond the spec** → nice to have, not a blocker
- **Entire file quality** when only reviewing a diff → stay scoped to what changed
- Do not request architecture changes in a code review — that belongs to a design/architecture pass
- Do not request refactoring for code NOT in the diff — stay in scope
- Do not flag style issues if there's a linter/formatter configured — trust the tooling

---

## Quick Reference

| Situation | Action |
|-----------|--------|
| Found a CRITICAL | Stop. Fix before anything else. |
| Found only MEDIUM/LOW | Ship with notes — document as TODOs |
| Unsure if it's a bug | Check with a failing test — if you can write one, it's a bug |
| Hard to review your own code | Use the checklist sequentially — don't free-form |
| Reviewing a large diff | Break it into logical chunks, review each separately |

---

## Review the Review
Before presenting findings:
1. Re-read your critique. Is each point actionable and specific, or vague?
2. Did you miss anything by focusing too much on one area?
3. Are you being performatively thorough (flagging non-issues to seem careful) or genuinely helpful?
4. Would a senior engineer agree with your top 3 findings?
Strip any finding that doesn't pass these checks.

### Architecture Step-Back
After reviewing individual code quality, zoom out:
- Does this change fit well with the existing architecture, or is it fighting it?
- If this pattern were extended to 10 similar cases, would the codebase be better or worse?
- Is there a simpler design that achieves the same goal?
These are LOW severity observations — note them and move on. But they compound into real architectural improvements over time.

---

## Second Opinion

After completing the review, consider running an independent reviewer for a different model's perspective — many CLIs expose a one-shot review command (e.g. a `review --uncommitted` or `review <diff>` subcommand). A second model often catches what the first missed.

---

## Skill Chain

| Stage | Skill |
|-------|-------|
| Before (writing the code) | **tdd-workflow** — tests first, then implementation |
| This skill | **code-review-session** — review before committing |
| Next (committing) | a git-workflow pass — conventional commit, right branch |
| If CRITICAL security found | **security-audit** — full audit if secrets or injection risk |
| If CRITICAL bug found | **debug-session** — diagnose root cause before fixing |
| After fix | **tdd-workflow** — add regression test for the bug found |
| For deep adversarial scrutiny | **devil-advocate** — stress-test the change against its own assumptions |

---

## Trigger Conditions

Use this skill when:
- You just wrote or modified code and want a pre-commit / pre-merge check ("review this code", "check my code", "look at what I wrote", "code review").
- You finished a feature or bug fix, or completed a tdd-workflow green phase.
- You are about to `git commit` a non-trivial change.

Do NOT use it for design reviews, architecture discussions, or whole-codebase audits — this skill reviews a **diff**, not a system.

## Out of Scope

- NOT for full codebase security audits — use `security-audit` instead
- NOT for diagnosing root cause of a bug — use `debug-session` instead
- NOT for architectural planning or design decisions — use `spec-driven-dev` or `decision-log` instead
- NEVER use this for reviewing an entire repo — scope to the diff only

## Common Traps

- **Reviewing code you haven't read**: Always read the file before commenting — reviewing a diff without understanding the surrounding code leads to incorrect or irrelevant suggestions.
- **Suggesting changes that conflict with project rules**: Check the project's own conventions and rules files first — the project may have hard rules that override general best practices.
- **Over-reviewing trivial changes (formatting, style) instead of logic/architecture**: Focus on substance — if a linter handles it, don't flag it. Spend review time on correctness, security, and design.
