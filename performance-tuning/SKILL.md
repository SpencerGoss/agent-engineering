---
name: performance-tuning
description: >
  Use when code is slow, a pipeline has a bottleneck, a job is taking too long, memory
  usage is too high, or you need to benchmark before and after an optimization. Trigger on:
  "this is slow", "optimize", "profile this", "this job takes too long", "pipeline bottleneck",
  "memory usage", "benchmark", "it's hanging", "takes forever", "needs to be faster", "latency",
  "why is this slow". Always measure before optimizing — no guessing.
license: MIT
compatibility: "Works across Claude Code, Codex, Gemini CLI, Copilot CLI, and any Agent Skills-compatible host. No runtime dependencies — pure prompt structure."
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

## Hard Rules

- **Never optimize code you haven't profiled.** Intuition about what's slow is wrong more often than it's right. Measure first, every time.
- **Verify correctness after every optimization.** Run the full test suite and confirm output matches the original within acceptable tolerance. A faster wrong answer is still wrong.
- **Record before/after numbers.** No "I think it's faster now." If you can't show a measured delta, the optimization isn't done.

---

# Performance Tuning

Measure first. Optimize second. Verify improvement with data.

The diagnostic workflow is: **measure → identify the bottleneck → optimize → benchmark**. If your host also exposes a profiling-tool skill (cProfile flags, memory profiler setup), use it alongside this workflow — this skill owns the method, that owns the tool syntax.

---

## Before Optimizing: Check if You Should

Premature optimization checklist — if ANY of these are true, stop and reconsider:

- [ ] The code runs fast enough for current usage (users aren't waiting)
- [ ] This code runs rarely (not in a hot path)
- [ ] You haven't profiled to confirm this is the bottleneck
- [ ] There's a simpler architectural fix (e.g., cache the result, reduce API calls)
- [ ] The code is hard to understand and optimizing will make it worse

Only proceed when: (1) you have a measured bottleneck and (2) it actually impacts the user.

---

## Step 1: Measure (Profile First)

### Python — cProfile
```bash
# Profile a script
python -m cProfile -s cumtime your_script.py

# Profile with output to file (for large reports)
python -m cProfile -o profile.out your_script.py
python -c "import pstats; p = pstats.Stats('profile.out'); p.sort_stats('cumulative'); p.print_stats(20)"
```

### Python — memory_profiler
```bash
pip install memory_profiler

# Add @profile decorator to the function
# Then run:
python -m memory_profiler your_script.py
```

### Python — line_profiler (line-by-line)
```bash
pip install line_profiler

# Add @profile decorator to the function
# Then run:
kernprof -l -v your_script.py
```

### Node.js
```bash
# Built-in profiler
node --prof your_script.js

# Analyze output
node --prof-process isolate-*.log > profile.txt
```

---

## Step 2: Read the Profile Output

**What to look for:**
- `cumtime` (Python) — total time spent in function including callees → find the top offenders
- `tottime` (Python) — time in function excluding callees → find the actual slow code
- High call counts on unexpected functions → loop-in-loop or unnecessary repeated work

**Hot-loop amplification:** Look for functions called thousands of times inside a tight loop (per frame, per row, per request, per event). One slow function called 10,000 times = 10,000x the pain. Pull invariant work out of the loop.

---

## Step 3: Optimize the Bottleneck

### Python — Common Patterns

**Vectorize with pandas/numpy instead of loops:**
```python
# SLOW: Python loop over DataFrame
for i, row in df.iterrows():
    result.append(row['price'] * row['qty'])

# FAST: Vectorized operation
result = df['price'] * df['qty']
```

**Chunk large DataFrames:**
```python
# Process in chunks instead of loading all data at once
for chunk in pd.read_csv('large_file.csv', chunksize=10000):
    process(chunk)
```

**Use sets for membership testing:**
```python
# SLOW: O(n) list lookup
if item in my_list:  # searches entire list

# FAST: O(1) set lookup
if item in my_set:
```

**Cache expensive computations:**
```python
from functools import lru_cache

@lru_cache(maxsize=128)
def expensive_calc(n: int) -> float:
    ...  # computed once, cached after
```

### Hoist invariant work out of hot loops

A common bottleneck in any tick-by-tick or row-by-row pipeline: recomputing a derived series on every iteration when it could be computed once before the loop.

```python
# SLOW: recompute the same indicator on every iteration
for i, row in df.iterrows():
    signal = compute_metric_for_one_row(row)

# FAST: pre-compute the entire series once, vectorized
df['metric'] = compute_metric_series(df['value'])  # once, vectorized
```

### Database / SQL query optimization

```python
# Add an index on frequently filtered columns
conn.execute("CREATE INDEX IF NOT EXISTS idx_entity_date ON records (entity_id, record_date)")

# Inspect the query plan to see what the engine is doing
conn.execute("EXPLAIN QUERY PLAN SELECT ...")  # SQLite; use EXPLAIN ANALYZE on Postgres
```

### Avoid per-row Python callbacks

```python
# SLOW: apply() calls a Python function per row
df['result'] = df.apply(lambda row: complex_calc(row), axis=1)

# FAST: vectorized numpy operation
df['result'] = np.where(df['value'] > threshold, df['a'], df['b'])
```

---

## Step 4: Benchmark Before and After

Always record results. No "I think it's faster now."

```markdown
## Performance Benchmark — [Feature/Script Name]
Date: YYYY-MM-DD

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Job runtime (full dataset) | 45s | 8s | 82% faster |
| Memory peak | 2.1 GB | 340 MB | 84% less |
| Rows processed/sec | 1,200 | 8,400 | 7x faster |
```

Commit this table alongside the change so the gain is part of the record, not folklore.

---

## Quick Reference

| Situation | Tool |
|-----------|------|
| "What's slow?" | Python: `cProfile -s cumtime` |
| "Why is memory high?" | Python: `memory_profiler` |
| "Which line?" | Python: `line_profiler` |
| "Node.js bottleneck" | `node --prof` |
| "pandas loop slow" | Vectorize: replace `df.apply(...)` / `iterrows()` with column ops |
| "SQL query slow" | `EXPLAIN QUERY PLAN` (or `EXPLAIN ANALYZE`) + add an index |
| "Tick/row loop too slow" | Pre-calculate invariant series before the loop |

---

## Trigger Conditions

Use this skill when:
- Code works correctly but runs too slowly, hangs, or "takes forever"
- A pipeline, batch job, or simulation has a bottleneck
- Memory usage is too high or grows over a long-running process
- Latency on a hot path needs to come down
- You need a before/after benchmark to justify an optimization

Phrases that trigger: "this is slow", "optimize", "profile this", "pipeline bottleneck", "memory usage", "benchmark", "it's hanging", "needs to be faster", "latency", "why is this slow".

---

## Out of Scope

- NOT for fixing bugs that produce wrong output — run a systematic diagnosis (use **debug-session**) for correctness issues; this skill is for code that works correctly but runs too slowly.
- NOT for refactoring code structure without a measured bottleneck — use **refactor-session** for structural improvements; this skill requires a profiled performance problem.
- NOT for slowness caused by missing or stale data rather than code — diagnose the data path first (use **debug-session**) if the problem isn't the code itself.

---

## Common Traps

- **Optimizing without profiling first:** Intuition about what's slow is wrong more often than right. Spending hours optimizing a function that accounts for 2% of runtime while the real bottleneck (database query, network call, serialization) goes untouched. Always profile before touching code.
- **Micro-benchmarks that don't reflect real workloads:** A function benchmarked in isolation with 100 items may behave completely differently with 1M items (cache misses, memory pressure, GC pauses). Benchmark with production-representative data sizes and shapes.
- **Optimization that breaks correctness:** Replacing a stable sort with an unstable one, caching a result that should be recomputed, or removing a "redundant" check that handled an edge case. Always run the full test suite after any optimization and verify output matches exactly.
- **Caching without an invalidation strategy:** Adding `@lru_cache` or a manual cache speeds things up until stale data causes wrong results. Every cache needs a clear invalidation policy — time-based, event-based, or size-based. Unbounded caches also leak memory over long-running processes.
- **Vectorization that changes floating-point results:** NumPy/pandas vectorized operations may use a different floating-point evaluation order than Python loops, producing slightly different results. For financial calculations or ML model reproducibility, verify that optimized output matches the original within acceptable tolerance.

---

## Skill Chain

| Stage | Skill |
|-------|-------|
| Before optimizing | Confirm there's a real bottleneck — profile first (this skill) |
| This skill | **performance-tuning** — measure → optimize → benchmark |
| After optimizing | **tdd-workflow** — verify behavior unchanged with tests |
| If introduced a regression | **debug-session** — diagnose what broke |
| Reviewing the optimized change | **code-review-session** — catch subtle correctness issues |
