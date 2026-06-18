---
name: debug-session
description: >
  Use when something is broken and the cause isn't obvious. Trigger on: "it's not working",
  "tests are failing", "the build is broken", "nothing is running", "something weird is
  happening", "it worked before", "I don't know what changed", "rollback this", any error,
  crash, unexpected output, or regression. Also trigger when an agent made autonomous changes
  that need auditing or reversing. NOT for design reviews or refactoring — use
  code-review-session or refactor-session instead. Never assume cause — always diagnose before fixing.
license: MIT
compatibility: "Works across Claude Code, Codex, Gemini CLI, Copilot CLI, and any Agent Skills-compatible host. No runtime dependencies — pure prompt structure."
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

## Hard Rules

- **High-stakes systems first:** Any issue touching irreversible side effects (live order execution, payments, production data writes, anything in a live-money or production financial system) is CRITICAL regardless of apparent severity. Stop and assess before proceeding.
- **Never assume cause — always diagnose before fixing.** State a hypothesis BEFORE changing any code.
- **Fix one thing at a time** — never bundle multiple fixes or refactor while fixing.

# Debug Session — Systematic Root-Cause Diagnosis

A disciplined diagnose-before-fix workflow. The goal is to find the *root* cause, not patch the first symptom, and to leave the codebase with a regression test plus a captured insight so the same bug can't recur.

## Routing: Debug vs. Browser/UI Testing

Before starting, route to the right approach:

| Situation | Use |
|-----------|-----|
| App won't start, build fails, crash on load | **This skill** (debug-session) |
| Runtime error, logic bug, wrong data | **This skill** (debug-session) |
| "UI isn't doing what I expect visually" | Browser-automation testing (Playwright) |
| "Verify a specific UI interaction works" | Browser-automation testing (Playwright) |
| "Take a screenshot of the app" | Browser-automation testing (Playwright) |
| "Why isn't my button/form/component working?" | Start here → if no code error found, hand off to browser-automation testing |

If a debug session isolates the problem to a UI behavior (not a code crash), stop and switch to browser-level investigation.

---

## Step 0: Step Back (before any diagnosis)

Before investigating the specific bug, categorize it:

1. **What category of failure is this?**
   - Data issue (wrong input, corrupt data, missing values, type mismatch)
   - Logic error (wrong condition, off-by-one, incorrect algorithm)
   - State bug (race condition, stale cache, incorrect initialization)
   - Environment mismatch (works locally, fails in CI; dependency version; config difference)
   - Integration failure (API changed, schema drift, timeout, auth expired)
2. **What general debugging principle applies?**
   - If data: validate inputs first, check the pipeline upstream
   - If logic: find the smallest reproducing case, binary-search for the breaking change
   - If state: add logging at every state transition, check for concurrent access
   - If environment: compare environments systematically (versions, configs, env vars)
   - If integration: test the external dependency in isolation first
3. **What's the most common cause for this category in this codebase?**
   (Check any local record of past bugs/known failure modes if one exists.)

This narrows the search space BEFORE you read code. The category determines your strategy.

---

## Step 1: Get the Situation

Before asking the user anything, run these immediately:
```bash
git diff                    # what changed in the working tree
git log --oneline -5        # recent commit history
```

> **Fallback:** If `git diff` fails (git not found, not a repo, or permission error), skip it and ask: "What changed since it last worked? Any recent file edits, installs, or commands?"

> **Prior-insight retrieval:** Before diagnosing, check whatever local notes exist (project notes, prior-decisions file, persistent insight log) for past learnings in this domain. Grep for keywords from the error (module names, error types, domain tags). If a past insight matches, state it: "Previously learned: [insight]. Checking if it applies here." This prevents re-discovering the same root cause.

Then collect only what you still can't answer from context:

1. **What broke?** (error message, wrong behavior, crash — exact output if possible)
2. **When did it last work?** (last commit, last action, "never worked")
3. **What changed?** — answer from `git diff` / `git log` first; only ask if unclear
4. **Can you reproduce it?** — run the failing command directly first; only ask if intermittent

If the user gives a vague "it's broken" — run git commands first, then ask only the questions you can't self-answer.

---

## Step 2: Triage — How Bad Is It?

| Severity | Definition | Action |
|----------|-----------|--------|
| 🔴 Critical | App won't start / data at risk / irreversible side effects affected | Stop everything, stabilize first |
| 🟠 High | Core feature broken, tests failing, build fails | Fix before new work |
| 🟡 Medium | Feature degraded, visual bug, non-critical test fail | Fix in current session |
| 🟢 Low | Minor cosmetic, edge case, nice-to-have | Log and schedule |

**High-stakes note:** Any issue touching live execution / payments / production writes = 🔴 Critical regardless of apparent severity. Stop and assess before proceeding.

---

## Step 3: Reproduce

Don't debug what you can't reproduce. Steps:

1. Run the exact failing command / interaction
2. Capture the full error output (not just the last line)
3. Confirm it's reproducible (not a fluke)
4. If intermittent → note the conditions that trigger it

If it can't be reproduced:
- Check if it was a one-time environment issue
- Check for race conditions or async timing issues
- Try a clean restart (clear cache, restart dev server, fresh virtual environment)

---

## Step 4: Isolate

Narrow down *where* the problem lives before touching code.

**Frontend / web UI checklist:**
- [ ] Capture browser console output via a browser-automation tool (agents can't open browser devtools directly)
- [ ] Network requests failing?
- [ ] Which component throws? (error-boundary message)
- [ ] Does it fail with mock data too, or only with live data?
- [ ] Does a clean dependency install + dev-server restart fix it?
- [ ] Any missing env vars?

**Backend / service checklist:**
- [ ] Full traceback — read the *bottom* of the stack trace first
- [ ] Which module raises? Is it your code or a library?
- [ ] Does it fail in isolation (unit test) or only in the full pipeline?
- [ ] Environment issue? (runtime version, virtual environment active, deps installed)
- [ ] Any `.env` vars missing or misread?
- [ ] If it's a financial/trading system: does it fail in paper/sandbox mode? (if so, don't test in live mode)

**An agent made changes and broke something:**
- [ ] Run `git diff` to see exactly what changed
- [ ] Run `git log --oneline -10` to find the last known-good commit
- [ ] Were there session notes describing what was being built?

---

## Step 5: Diagnose

**Root cause, not symptoms.** Don't fix the first thing that looks wrong. Ask "Why did this happen?" at least twice. The first answer is usually a symptom. Infrastructure bugs especially tend to have a structural cause that recurs if you only patch the symptom. (Classic example: a series of "contamination" fixes that were all symptoms of one structural problem — a shared working directory.)

### Step 5a: Gather Diagnostic Evidence

IMPORTANT: Log your hypothesis BEFORE running reproduce attempts. Without a hypothesis first, you're guessing — not debugging.

Before forming a hypothesis, collect raw evidence:
- **Add temporary logging:** `console.log('DEBUG:', variable)` / `print(f"DEBUG: {variable}")` at the point of failure
- **Log raw API response shape:** Don't assume the shape — log `JSON.stringify(response)` or `print(response)` and read the actual output
- **Check async/await coverage:** Scan the call chain — every async function needs an `await`; a missing one returns a Promise/coroutine object instead of the value
- **Verify immutability:** If state looks wrong in a UI framework, check for direct mutation (`state.x = y` instead of an immutable update)
- **Check if the bug exists in isolation:** Comment out surrounding code — does the issue persist with minimal input?

Only after collecting this evidence, proceed to form a hypothesis.

Form a hypothesis *before* writing any fix. State it explicitly:

> "I think the issue is [X] because [evidence Y]. If I'm right, fixing [Z] should resolve it."

Evidence that supports a hypothesis > instinct. Common root causes by type:

**Import / dependency errors:**
- Missing package → install it
- Version mismatch → check the dependency manifest (`package.json` / `requirements.txt` / lockfile)
- Circular import → restructure the module

**Config / env errors:**
- Missing `.env` var → add it
- Wrong path → verify by printing/logging it
- Wrong port / host → check the config file

**Logic errors:**
- Off-by-one, wrong condition, inverted boolean
- Async timing (promise not awaited, callback order)
- State mutation (state directly mutated instead of via the framework's setter)

**Data errors:**
- API response shape changed
- Null/undefined where a value was expected
- Type mismatch (string vs number)

**Agent-introduced errors:**
- Read the diff — what did it actually change vs what it said it changed?
- Did it touch a file it shouldn't have? (cross-reference project rules / off-limits paths)
- Did it change shared utilities, breaking other things?
- If an agent may have caused the bug, read the project's "Off-Limits" / "Hard Rules" sections first — these contain constraints agents sometimes miss.

---

## Step 6: Fix

Only after Steps 3–5 are complete.

**Fix rules:**
- Fix one thing at a time — don't bundle multiple fixes
- Make the smallest change that addresses the root cause
- Don't refactor while fixing — add that to a TODO
- Add a comment if the fix is non-obvious: `# Fixes: [what was wrong and why]`

**For high-stakes (trading/financial/production) fixes:**
- Test in paper/sandbox mode first, always
- If fixing execution logic: write a unit test before applying the fix
- Never go live to "test" a fix

---

## Step 7: Verify

After the fix:

1. Reproduce the original failure → confirm it no longer occurs
2. Run the test suite: does anything new break?
3. Spot-check adjacent behavior — did the fix have side effects?
4. Commit with a clear message: `fix: [what was broken and what fixed it]`

**Don't mark as resolved until verified.**

---

## Step 8: Capture

After resolving, capture what happened so it doesn't repeat:

1. **Add a project rule** (in your agent-instructions file, e.g. `CLAUDE.md` / `AGENTS.md`) if an agent was the cause or this is a common mistake to avoid
2. **Log it** in your project changelog/journal: what broke, what caused it, how it was fixed
3. **Add a test** if the bug is reproducible and no test would have caught it
4. **Update `.claude/rules/`** (or equivalent rules dir) if there's a pattern to prevent
5. **Persist the pattern** to whatever cross-session memory you keep so future sessions don't repeat the diagnosis. Format: `"[BUG PATTERN]: [cause] → [fix]"` — e.g., `"[API KeyError]: response shape changed in v2 → access data['result']['price'], not data['price']"`
6. **Extract a compact insight** to your persistent notes, tagged with the relevant domain. Before writing, scan existing entries for the same tag — consolidate rather than duplicate. Format:
   ```
   [DATE] [tag] INSIGHT: [root cause in one sentence]
   [DATE] [tag] WHY: [why it wasn't obvious / what made it tricky]
   ```
   Tag with the domain (e.g., `[data]`, `[ml]`, `[api]`, `[auth]`) for future retrieval.

**Was the agent the cause?** Ask explicitly: "Did the agent produce the bug (wrong edit, wrong approach, wrong assumption)?" If yes, record the mistake pattern, check whether it's a repeat, and promote repeats into your hard-rules file so they can't recur.

---

## Platform-Specific Checks (when on Windows)

- Encoding: prefix Python scripts with `PYTHONUTF8=1` (cp1252 vs UTF-8 mismatches)
- Paths: use forward slashes or raw strings; never backslash in Python imports
- Signal handlers: use `try/except`, not `add_signal_handler` (not supported on Windows)
- Shell: verify `which bash` works; PowerShell behaves differently for shell scripts

---

## Quick Reference: Common Errors

### Frontend / web UI
| Error | Likely Cause | Quick Fix |
|-------|-------------|-----------|
| White screen / blank render | UI framework crash — check console | Capture console output via browser automation |
| `Cannot read properties of undefined` | Data not loaded before render | Add loading state / null check |
| Styles missing | Utility class not in config | Check class name, check the CSS/build config |
| API 404 | Wrong endpoint or dev server not running | Confirm the dev server is running |
| Stale UI after change | Hot reload failed | Hard refresh (Ctrl+Shift+R) |

### Backend / service
| Error | Likely Cause | Quick Fix |
|-------|-------------|-----------|
| `ModuleNotFoundError` | Package not installed | Install it / activate the correct virtual environment |
| `KeyError` on API response | API response shape changed | Print the raw response and inspect |
| Logic not triggering | Condition / branch not met | Add debug logging where the condition is evaluated |
| Request rejected by an external service | Input exceeds a limit | Check the relevant limit/config value |
| `AttributeError` on config | Missing key or wrong default | Check the config module and class defaults |

> **Bug appears in two repos at once:** Likely a shared dependency. Debug each repo independently first; check shared tooling (runtime, package manager, git, virtual environment) last.

---

## Rollback Procedure

If changes broke things and you want to revert:
```bash
# See what changed
git diff HEAD

# Unstage (keeps working-tree changes)
git reset HEAD

# Discard all working-tree changes
git reset --hard HEAD

# Revert a specific file only
git checkout HEAD -- path/to/file.py

# Go back to a specific commit
git log --oneline -10     # find the commit hash
git reset --hard <hash>   # go back to it
```

Always confirm with the user before running `git reset --hard` — it discards changes permanently.

---

## Second Opinion

If the bug is elusive after 2+ hypotheses, suggest a different model's perspective (e.g. `codex exec 'Find the bug in this diff'`). If the bug involves an external API/library behaving unexpectedly, suggest verifying the documented behavior with a grounded search tool (e.g. `gemini -p 'Does [API/function] actually work this way? Check the docs.'`).

---

## Trigger Conditions

- "It's not working", "tests are failing", "the build is broken", "nothing is running"
- "Something weird is happening", "it worked before", "I don't know what changed"
- "Rollback this" / any request to audit or reverse autonomous agent changes
- Any error, crash, unexpected output, or regression where the cause isn't obvious

## Out of Scope

- An agent produced structurally wrong *output* (missed requirements, ignored instructions, added unwanted scope) → that's a reasoning bug, not a code bug; analyze the prompt/response instead
- "You missed the point", "that's not what I asked" → reasoning analysis, not this skill
- Design reviews → use code-review-session
- Refactoring → use refactor-session
- This skill is for CODE bugs (crashes, test failures, broken builds), not model-reasoning failures.

## Common Traps

- **Chasing symptoms instead of root cause:** A `TypeError` on line 50 may be caused by bad data on line 12. Trace the data flow backward from the error site before fixing at the crash point.
- **Tests pass but prod fails:** Tests often use mocked data or a clean environment. Bugs that only appear with real data, stale cache, or platform-specific behavior (encoding, timezone) won't be caught by unit tests. Reproduce in the closest-to-prod environment available.
- **Fixing two things at once:** Bundling a "quick cleanup" with a bug fix makes it impossible to tell which change resolved the issue — and if it regresses, you can't bisect cleanly.
- **Assuming the error message is the bug:** Error messages describe *what* failed, not *why*. "Connection refused" might be a missing env var, not a network issue. Read the full stack trace and check preconditions before trusting the message.
- **Skipping diagnostic evidence (Step 5a):** Jumping to a hypothesis without logging actual values leads to "I thought it was X" debugging loops. Always add temporary logging and read real output before forming a theory.
