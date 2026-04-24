---
name: tdd-workflow
description: >
  Use when a spec or plan exists and it's time to write code — or when fixing a bug. Trigger on:
  "fix this bug", "write tests for", "write a function that", "create a module", "the spec is
  ready, let's implement", or any request to write working code when the WHAT is already clear.
  The hard gate: no implementation before a failing test exists.
license: MIT
compatibility: Works across Claude Code, Codex, Gemini CLI, Copilot CLI. Assumes a test runner (pytest, Jest, Vitest, or language equivalent), git, and a shell.
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

# TDD Workflow

Write the test first. Watch it fail. Write minimal code to pass it. Refactor.

**Hard gate:** Never write implementation code before a failing test exists. No exceptions.

---

## The Cycle

### RED: Write a Failing Test

1. Write a test that describes the behavior you want
2. Run it — it MUST fail (if it passes, the test is wrong or already implemented)
3. Commit: `test: add failing test for <feature>`

```python
# Python example
def test_calculate_spread():
    result = calculate_spread(home_score=105, away_score=98)
    assert result == 7  # RED: function doesn't exist yet
```

```javascript
// JavaScript/TypeScript example
test('calculateSpread returns point difference', () => {
  expect(calculateSpread(105, 98)).toBe(7);  // RED: not implemented
});
```

### GREEN: Write Minimal Implementation

1. Write the **smallest possible code** that makes the test pass
2. Run — it MUST pass
3. No extra code, no "while I'm here" additions
4. Commit: `feat: implement <feature>`

If GREEN fails: your implementation is incomplete. Debug before moving to REFACTOR.
Never skip RED → always verify test fails before implementation.
Never modify tests to make them pass — fix the implementation.

```python
def calculate_spread(home_score: int, away_score: int) -> int:
    return home_score - away_score  # Minimal. Done.
```

### REFACTOR: Clean Up Without Breaking

1. Improve names, remove duplication, simplify logic
2. Run tests — still must pass
3. Commit: `refactor: clean up <feature>`

### EXTRACT: Capture What You Learned (Optional but High-Value)

After REFACTOR, if you discovered anything non-obvious during the cycle:
- A constraint the test revealed about the system
- An approach that failed during GREEN and why
- A pattern that applies beyond this specific function

Write it to your project notes using your persistent insight format. Before writing, scan existing notes for the same tag — MERGE or UPDATE if an entry already exists, do not add a duplicate. Duplicate domain insights dilute the core insights and cause confusion.

Skip if nothing surprising happened. Don't force it.

---

## Per-Language Quick Start

### Python (pytest)
```bash
# Install
pip install pytest
uv pip install pytest  # if using uv

# Run tests
pytest                          # all tests
pytest tests/test_module.py     # specific file
pytest -k "test_name"           # specific test
pytest --tb=short               # short traceback
```

### JavaScript/TypeScript (Jest or Vitest)
```bash
# Run tests
npm test                        # Jest
npx vitest                      # Vitest
npm test -- --watch             # watch mode
npm test -- --coverage          # coverage report
```

---

## Coverage Gate

Before marking any feature complete:
- Run coverage check
- Must reach 80%+ on new code
- If below: write more tests for uncovered branches

```bash
# Python
pytest --cov=. --cov-report=term-missing

# JavaScript (Jest)
npm test -- --coverage
```

---

## "Already Have Code" Path

If you're writing tests for existing code (characterization tests):

1. Write tests that describe what the code **currently does** (not what it should do)
2. Run — they should pass (capturing existing behavior)
3. Commit: `test: add characterization tests for <module>`
4. Now refactor or fix — tests will catch regressions

**Legacy Code Exception:**
If modifying code that has no existing tests and is too complex to test from scratch:
1. Write characterization tests first (describe current behavior, even if broken)
2. These are not RED tests — they document the existing contract
3. Then apply TDD for new behavior on top of characterized behavior
4. Do NOT skip characterization tests and go straight to new tests on legacy code

---

## Quick Reference

| Situation | What to do |
|-----------|-----------|
| Tempted to write code first | Stop. Write the test first. |
| Test passes immediately | The test is wrong — check your assertion or the code already exists |
| Can't figure out what to test | Write the function signature in a comment, then write what the caller expects |
| Test is too hard to write | The function is too large — break it into smaller pieces |
| "Just a simple function" | Still write the test first. Simple tests take 60 seconds. |
| Feature has multiple behaviors | One test per behavior. Don't test everything in one test. |

---

## Rules

- **One assertion per test** when possible (makes failures clear)
- **Test names describe behavior:** `test_returns_none_when_input_empty` not `test_function`
- **Arrange-Act-Assert pattern:** set up → call function → assert result
- **No implementation before RED:** if you wrote code before a failing test, delete the code and start over
- **Commit each phase:** RED commit, GREEN commit, REFACTOR commit — makes history readable

---

## Out of Scope

- NOT for planning what to build — use a spec-driven-development skill to write the spec before starting TDD
- NOT for diagnosing why something is broken — use a debug skill to find the root cause first, then write a fix test here
- NOT for code review or style enforcement — use a code-review skill after the RED/GREEN/REFACTOR cycle
- NEVER use this for writing tests against code you don't understand yet — read the code or spec first

---

## Common Traps

- **Testing implementation details instead of behavior:** Tests that assert internal method calls, private state, or execution order break on every refactor. Test the *output* given an *input* — not how the function arrives at the answer.
- **Mocking too much:** When every dependency is mocked, your test proves the mocks work, not the code. Mock external services (APIs, databases) but let internal modules interact naturally. If a test requires 5+ mocks, the function under test likely needs to be decomposed.
- **GREEN test that passes for the wrong reason:** A test that asserts `result is not None` passes even when the result is garbage. Write assertions that would fail if the implementation returned a plausible-but-wrong value (e.g., assert exact value, not just truthiness).
- **Skipping RED — writing code before the failing test:** If you write the implementation first, the test might pass because it's testing what you wrote rather than what's correct. The RED step catches this: if the test passes immediately, something is wrong.
- **Flaky tests from shared state or timing:** Tests that depend on execution order, global variables, or wall-clock time pass locally but fail in CI. Use fresh fixtures per test (`scope="function"` in pytest), avoid `time.sleep()` in assertions, and never depend on test ordering.
