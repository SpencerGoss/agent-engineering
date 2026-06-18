# Pattern: Adversarial Multi-LLM Council

A protocol for getting a *genuinely* diverse second opinion from multiple LLMs —
and the frame-isolation trick that stops a "council" from collapsing into expensive
agreement. Built for high-stakes decisions, design reviews, and fact-checks where a
single model's blind spots are a real risk.

## Two primitives

The council is two complementary shapes, not one:

**1. Parallel fan-out (diverse priors on one question).** Several models answer the
*same* question independently; a chair synthesizes. Use when you want redundancy and
debate — multiple priors on a decision.

**2. Sequential chain (adversarial verification of a claim).** A *grounding* model
gathers evidence → a *critic* model audits that evidence for overreach → a
deterministic rule emits the verdict. This is a primary/critic/verifier shape, and
it fails **closed**: if the critic's output is truncated or the two models disagree,
the verdict is "unverified," never a false "true."

Route by intent: fan-out for "which option is better?"; chain for "is this claim
actually true?"

## The failure mode this defeats

Run several copies of capable models on the same prompt and they tend to **converge**
— not because the answer is right, but because they share framing, training, and
therefore blind spots. The debate *feels* robust while adding nothing. Theory predicts
exactly this: in a shared-knowledge regime, the marginal value of debate trends toward
zero (arXiv:2603.05293); under shared framing, more rounds inherit the same failure-mode
coverage as one round.

Consensus is therefore **not** a quality signal. A council that only measures
agreement is measuring the wrong thing.

## Frame isolation — the load-bearing move

The fix is not more models or more rounds. It is **breaking the shared frame**:

- **Premise audit.** Before taking the vote, dispatch one model *in parallel* on a
  deliberately divergent prompt: *"Attack the premise. Do NOT answer the question."*
  Crucially, this lane is **frame-isolated** — it does **not** receive the project
  context or the framing the normal lane saw, so it cannot be anchored to the same
  assumptions. Its output is labelled *"premise audit — not a vote"* and never enters
  the tally. It exists to surface the question nobody asked.

- **Three mandatory attacks** the premise audit always runs: *(a)* Is action required
  at all — what's the evidence that doing nothing is correct? *(b)* Is the framing of
  the problem itself wrong? *(c)* What signal or safeguard is *lost* if the proposed
  change ships? These three catch the class of failure where a smooth, well-argued
  council ships something that rebuilds existing infrastructure or bakes in a
  methodology bias.

- **Adversarial framing for convergence.** Before declaring "we agree, ship it" on a
  load-bearing decision, run exactly one pass with explicit *"find the worst attack"*
  framing. Agreement rounds share a cone of attention; one adversarial pass reliably
  surfaces gaps that N agreement rounds systematically miss — at the cost of a single
  extra dispatch.

## A governance gate

Diverse opinions are only useful if a *blocking* one can actually stop the train. A
lightweight protocol: each model emits a structured vote (`stance`, `reason`,
`tier`); the parser **fails closed** on a malformed or missing high-stakes vote; and a
quorum of blocking votes mechanically halts the action until explicitly overridden
with a logged reason. This converts "the council had concerns" from prose that
evaporates into an auditable stop.

## Cost discipline

A premise audit is one extra model call (~30–60s). The failure it prevents —
shipping a confidently-wrong, agreed-upon decision — is far more expensive. But
councils are not free: 3–7 participants is the sweet spot, the lever is **framing
heterogeneity, not participant count**, and the right trigger to *expand* a council
is a demonstrated miss, not merely available capacity.

## References

- Debate value in shared- vs. compositional-knowledge regimes — arXiv:2603.05293
- Information-asymmetric debate — arXiv:2311.08702, arXiv:2409.16636
- Closed-system reasoning bound — arXiv:2605.01704
