# Pattern: Multi-Agent Orchestration (fan-out with independent verification)

A practical pattern for decomposing work across many LLM agents **without** the
quality collapse that naive "let the agents debate" approaches suffer from. It is
the loop topology behind coverage sweeps, competing-design tournaments, large
migrations, and any task too big for one context window.

## The shape

A **supervisor** classifies the *shape* of the task and dispatches the matching
topology. It does not do the work itself: it decomposes, fans out, and synthesizes.

| Task shape | Topology |
|---|---|
| Coverage / breadth (audit N files, find-all-X) | parallel **finders** → **verifier** per finding → act |
| Wide solution space (no obvious winner) | N **attempters** (diverse framings) → **judge** panel → synthesize |
| Decomposable implementation (N independent changes) | parallel **fixers** in isolated worktrees → **verifier** |
| Is-this-real? (a finding, a design) | independent **verifiers** (different model / lens) |
| Grounded research | **researchers** by angle → **synthesizer** |
| Single-stream dependent work | **solo**: coordination overhead exceeds the benefit |

The unit of reuse is a **role** (finder / verifier / fixer / researcher /
synthesizer), filled by a **domain specialist** where expertise matters. Role and
domain are orthogonal: pick the shape, then staff it with the right expert.

## Why a *verifier*, not a vote

The central design decision: the verifier checks the **work** (re-derives the
claim, re-reads the file, attacks the finding), it never asks *"do the agents
agree?"* Agreement is not evidence. Copies of one model prompted to debate
produce **diverse phrasings of one perspective**, not diverse perspectives, a
result with information-theoretic backing (the "Reasoning Trap," arXiv:2605.01704)
and observed repeatedly in practice: multi-agent debate can preserve answer
accuracy while *degrading* the reasoning behind it.

So the cure for low-quality fan-out is **not more rounds**. It is **independent,
adversarial verification**:

- Verify by re-execution, not by consensus.
- Default the verifier to *reject* on uncertainty ("confirm from the file itself
  or return false").
- Where a finding can fail in more than one way, give each verifier a **distinct
  lens** (correctness / security / does-it-reproduce) instead of N identical
  checkers. Diversity catches failure modes redundancy cannot.

A measured example from production use: a finder pass surfaced 10 candidate
findings; an independent verifier discriminated them to *3 real / 5 maybe / 2
rejected*, catching a hallucinated citation and a non-reproducing edge case. The
verifier earned its cost precisely because it was not a rubber stamp.

## The six coordination rules

Every fan-out obeys these, or it degrades into noise:

1. **Stateless schema-returns.** Each agent returns a validated structured object,
   not free prose. The orchestrator composes results; agents don't share mutable state.
2. **No two agents own one file.** Parallel writers on the same file is the #1
   failure mode. Partition by file/directory ownership, or isolate in worktrees.
3. **Every loop has a goal-verifier AND a hard stop.** A finish-line check *and* a
   bound (max count / token budget / "K consecutive empty rounds"), never an
   open-ended "keep improving."
4. **Spec before fan-out.** Write the objective, constraints, and output contract
   once; hand the same spec to every worker. The spec is the coordination, not a
   chat history they all read.
5. **Route cheap vs. strong.** Mechanical stages get a cheap model; the hardest
   verify/judge stages get the strong one. Don't pay frontier rates for grunt work.
6. **Prove one worker, then the swarm.** Run a single worker end-to-end and confirm
   the shape before spawning fifty.

## Bounding (the anti-runaway discipline)

Unbounded agent loops are how you light money on fire. Three bound styles:

- **Loop-until-count**: accumulate to a target (`while found < N`).
- **Loop-until-dry**: for unknown-size discovery, stop after *K consecutive
  rounds* surface nothing new (a simple `count < N` misses the long tail).
- **Loop-until-budget**: scale depth to a token target; once spent, stop.

And **never silently cap**: if you bound coverage (top-N, no-retry, sampling), say
so in the output. Silent truncation reads as "covered everything" when it didn't.

## When NOT to fan out

Roughly 80% of line-by-line coding is single-stream dependent work where one agent
is correct and coordination overhead is pure cost. Fan-out is for the
coverage / exploration / decomposable / adversarial shapes. The sweet spot is
**3 to 7 workers** per fan-out: marginal gains plateau past 7. Don't manufacture a
workflow to look sophisticated.

## References

- *The Reasoning Trap: An Information-Theoretic Bound on Closed-System Multi-Step
  LLM Reasoning*, arXiv:2605.01704
- *MARBLE: Multi-Agent coordination via graph topologies*, arXiv:2503.01935
- Debate / information-asymmetry results: arXiv:2603.05293, arXiv:2311.08702, arXiv:2409.16636
