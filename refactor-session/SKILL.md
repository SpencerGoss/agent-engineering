---
name: refactor-session
description: >
  Use when improving code structure without changing behavior — cleaning up messy code,
  extracting functions, renaming for clarity, removing duplication. Trigger on: "refactor this",
  "clean up this code", "this is too messy", "extract this function", "this function is too
  long", "rename this", "remove duplication", or any structural improvement task. Hard rule:
  never refactor and change behavior in the same commit.
license: MIT
compatibility: Works across Claude Code, Codex, Gemini CLI, Copilot CLI. Assumes git, a test runner, and grep (or equivalent) are available.
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

# Refactor Session

Change structure, not behavior. If behavior changes, that's a feature or fix — not a refactor.

**Hard rule:** Never refactor and change behavior in the same commit. Pick one.

---

## Think Before Refactoring
Before jumping into structural changes:
1. **What is the END STATE you want?** Describe the ideal structure, not just "clean up this mess."
2. **Are there multiple valid end states?** Which serves the project best long-term?
3. **Is this the right time?** Does this refactor block other work, or is it investment for the future?

Don't start moving code until you can describe where everything should end up.

**Rule of Three before abstraction:** Count concrete use cases RIGHT NOW (not hypothetical). If < 3, write the concrete version. Three similar lines of code is better than one premature abstraction. Only extract when you have 3+ real instances.

---

## Before Starting: Get a Safety Net

1. **Write characterization tests** — capture what the code currently does
2. **Run tests** — they must pass before you start
3. **Commit** the tests: `test: add characterization tests for <module>`

Now any breakage is immediately visible.

---

## Refactor Types

| Type | When to use | Example |
|------|-------------|---------|
| **Extract function** | Block of code does one thing, deserves a name | 20-line loop → `calculate_total()` |
| **Rename** | Name doesn't describe what it actually does | `data` → `player_stats_by_game` |
| **Remove duplication** | Same logic in 3+ places (rule of three) | Extract to shared utility |
| **Simplify condition** | Complex boolean → named predicate | `if score > 0 and game_played` → `if is_valid_game(score, game_played)` |
| **Split large function** | Function does more than one thing | Split at natural seams |
| **Flatten nesting** | More than 3 levels deep | Early returns, guard clauses |

---

## The Process: One Change Per Commit

```
1. Make ONE structural change
2. Run tests — must still pass
3. Commit: "refactor: <what changed>"
4. Repeat
```

Never batch multiple refactors. One change = one commit = easy to revert if needed.

---

## Guard Clauses: Flatten Deep Nesting

```python
# BEFORE: hard to read with deep nesting
def process(data):
    if data:
        if data.is_valid():
            if not data.is_processed:
                do_work(data)

# AFTER: guard clauses (same behavior, flat structure)
def process(data):
    if not data:
        return
    if not data.is_valid():
        return
    if data.is_processed:
        return
    do_work(data)
```

---

## Rename Safely

Rename one thing at a time. After renaming, verify all usages updated:
```bash
grep -r "old_name" .    # find any remaining references
```
Fix every usage before committing. IDE refactor tools are safer than manual find-replace.

---

## When to STOP

Stop immediately if:
- **You find a bug** → fix it in a separate `fix:` commit, then resume
- **You want to add a feature** → do it in a separate `feat:` commit
- **Tests start failing and you're not sure why** → `git diff` to see what changed, revert the last step
- **The refactor keeps growing** → stop, open a TODO, scope it properly first

---

## Quick Reference

| Situation | What to do |
|-----------|-----------|
| Found a bug while refactoring | Stop. `git stash` refactor, fix bug, then restore. |
| Tests fail after rename | `grep -r "old_name" .` — find missed references |
| Function getting complex | Extract smaller pieces before touching names |
| Unsure if behavior changed | Your characterization tests will tell you |
| Refactor feels like a rewrite | Stop — you're redesigning, not refactoring |
| Tempted to add a feature | Open a TODO. Finish the refactor first. |

---

## Out of Scope

- NOT for fixing bugs or changing behavior — use a debug skill for bugs, a TDD skill for new behavior
- NOT for adding new features while restructuring — use a spec-driven-development skill for feature work
- NEVER use this for full rewrites or redesigns — if the refactor feels like a rewrite, scope it as a new feature instead
- NOT for code review of someone else's work — use a code-review skill instead

---

## Common Traps

- **Refactoring without tests**: Run existing tests first to establish baseline — without a green test suite, you can't tell if your refactor broke behavior.
- **Scope creep during refactoring ("while I'm here..."):** Stick to the stated refactoring goal — mixing refactoring with feature work or unrelated cleanup creates hard-to-review commits and increases revert risk.
- **Breaking public API contracts**: Check for callers before renaming/removing — grep for all usages of any function, class, or method you're changing. External consumers won't get your rename for free.
