---
name: judge-panel
description: Use when a task has a WIDE solution space and one-attempt-iterated is likely to lock in a mediocre first idea — "what's the best way to structure X", "try a few approaches to Y and pick the best", "I'm not sure which design wins", "compare a few implementations", competing-implementation or competing-design forks where the right answer isn't obvious up front. NOT for narrow tasks with one obvious approach (just do it), NOT for stress-testing a single proposal (use devil-advocate).
license: MIT
compatibility: "Works across Claude Code, Codex, Gemini CLI, Copilot CLI, and any Agent Skills-compatible host. No runtime dependencies — pure prompt structure."
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

## Hard Rules
- **Angles are load-bearing.** Every parallel attempt gets the SAME task but a DIFFERENT strategy angle. If you give all attempts an identical prompt, you get N rephrasings of one idea — the agreement-collapse failure this skill exists to defeat.
- **Judges must be independent of authors.** An attempt's own author may never score it. A judge that shares an author's reasoning shares its blind spots.
- **Score concrete, named dimensions — never "which is better?"** Vague rubrics produce noise. Define 3–6 scorable criteria up front.
- **Synthesize; do not just crown the top score.** Graft the specific strengths judges flagged in the runners-up. Discarding them throws away half the value.
- **Don't use this on narrow tasks.** It burns N× the tokens. Reserve it for genuinely wide forks where a wrong early commit is expensive.

# Judge Panel (parallel attempts → independent judging → synthesis)

A structured way to beat the "first idea, iterated" trap on wide-open problems. Instead of committing to one approach and polishing it, you generate several genuinely different attempts in parallel, judge them against explicit criteria, and synthesize from the winner — grafting the best ideas from the runners-up. Grounded in the multi-agent finding that **diversity of attempt is the dominant driver of quality**: heterogeneous attempts beat homogeneous ones, and correlated errors among same-prompt agents collapse the gains.

## When this beats just doing it
- The solution space is wide and the best shape is genuinely unclear (data schema, API surface, algorithm choice, refactor strategy, page layout).
- A wrong early commit is expensive to undo.
- You can state what "good" looks like as concrete, judgeable criteria.

If none of those hold, skip this — generating several attempts at an obvious task is waste.

## Step 1 — Define the contest (do this in your main working context, before spawning anything)
1. **The task**: one precise sentence of what each attempt must produce.
2. **The angles**: name 3–5 *genuinely different* strategies to seed diversity (e.g. "MVP-first", "performance-first", "maintainability-first", "use-an-established-library", "from-scratch"). Same task to all + a different angle each — that's what breaks agreement-collapse. Homogeneous attempts defeat the purpose.
3. **The judging criteria**: 3–6 concrete, scorable dimensions (correctness, simplicity, extensibility, performance, test surface). Vague criteria → vague judging.

## Step 2 — Run the tournament
Three phases: **Attempts → Judge → Synthesize**.

**Attempts (parallel).** Spawn one sub-agent per angle. Each gets the identical task plus its assigned strategy:

> Produce an attempt at: `<TASK>`. Use the "`<ANGLE>`" strategy specifically. Return the concrete artifact plus a short rationale.

If your host has a parallel-agent or workflow primitive, fan these out concurrently. If it doesn't, run them sequentially in fresh contexts — the key invariant is that each attempt is produced independently, without seeing the others (so they don't converge).

**Judge (parallel, independent).** Each completed attempt is scored by a *separate* agent against the same rubric — independence matters more than redundancy:

> Score this attempt against these criteria: `<CRITERIA>`. Be specific and critical. Return a per-criterion score, a total, and the single strongest and weakest aspect.
>
> ATTEMPT:
> `<the artifact>`

Have each judge return structured output so the results are sortable, e.g.:

```json
{
  "scores": { "<criterion>": 0 },
  "total": 0,
  "strongest": "...",
  "weakest": "..."
}
```

**Rank.** Sort the judged attempts by total score, descending. Carry every judge's `strongest`/`weakest` notes forward into synthesis — they are the cross-pollination signal.

## Step 3 — Synthesize (main context)
Read the ranked results. Do NOT just take the top score blindly:
- Adopt the winner as the base.
- **Graft** specific strengths the judges flagged in runners-up (the panel's value is cross-pollination, not just ranking).
- If the top two are within a hair, name the tradeoff explicitly and pick with a one-line justification.
- Implement the synthesized result in your main context, then verify normally (tests / run).

## Trigger Conditions
- "What's the best way to structure X" where X has many viable shapes.
- "Try a few approaches to Y and pick the best."
- "I'm not sure which design wins" / "compare a few implementations."
- Competing-implementation or competing-design forks where the right answer isn't obvious up front and a wrong early commit is costly.

## Out of Scope
- Narrow implementation with one obvious approach → just implement it (optionally plan it first with `spec-driven-dev`).
- Adversarial stress-test of a single proposal → `devil-advocate`.
- A focused review of code already written → `code-review-session`.
- Verifying one discrete factual claim → that's fact-checking, not a design tournament.

## Common Traps
- **Homogeneous attempts** — if all agents get the same prompt with no distinct angle, you get N phrasings of one idea (the agreement-collapse failure mode). The *angles* are load-bearing.
- **Self-judging** — never let an attempt's own author score it; judges must be independent. A model judging its own work shares its blind spots (the correlated-error failure).
- **Vague rubric** — "which is better?" yields noise. Score concrete, named dimensions.
- **Over-use** — this burns N× tokens. Reserve for genuinely wide forks; for narrow tasks it's pure overhead.
- **Skipping synthesis** — taking the raw winner and discarding the runners-up throws away half the value. Graft.
- **Attempts seeing each other** — if the parallel attempts can read one another mid-generation, they converge and you lose the diversity you paid for. Keep them isolated until judging.

## Skill Chain
| After this | Next |
|---|---|
| Winner synthesized | `spec-driven-dev` (if it's a feature) or implement inline |
| Implementation done | `code-review-session` |
| Surprising winner / insight | capture the insight for future sessions |
