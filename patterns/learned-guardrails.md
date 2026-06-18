# Pattern: Learned Guardrails (an "immune system" for an agent)

A pattern for making an AI coding agent *stop repeating its own mistakes*. Instead of
hoping the model "remembers" a past failure, you codify each failure into a structured
guardrail and **enforce it mechanically** at the moment the mistake would recur. The
biological metaphor is an immune system: an exposure produces an antibody, and the
antibody fires before the same pathogen lands again.

## The problem it solves

LLM agents are amnesiac about their own errors. The same class of mistake —
swallowing an exception, hardcoding a secret, deleting a file the user didn't ask to
delete, "fixing" a flaky test by weakening the assertion — recurs across sessions
because nothing carries the lesson forward in a way that *acts*. Prose post-mortems in
a doc don't fire; they're read once and forgotten.

## The shape

**1. Capture.** When a mistake is corrected, write a structured record, not a
narrative: a stable `id`, the `pattern` (what went wrong), the `trigger` (the
mechanically-detectable signature), a `severity`, and the `defense` (what to do
instead). One fact per record.

**2. Recall.** Before acting in a domain a record covers, the relevant guardrails are
surfaced into the agent's working context — so the defense is present *at decision
time*, not buried in a doc the agent won't open.

**3. Enforce (the load-bearing part).** Where a trigger is mechanically detectable,
wire it into a pre-action hook that **blocks** the action and feeds the reason back —
not a warning the agent can ignore, an actual stop. The ones that can't be mechanically
detected stay as recall-time guidance.

The hard-won lesson: *capture and recall without enforcement is theater.* A guardrail
that merely logs while the bad action still happens provides false comfort. The value
appears only when the trigger can actually halt the action and surface why.

## Keeping it honest (avoiding autoimmunity)

Two failure modes turn an immune system against its host, and both need explicit
defenses:

- **Over-broad triggers (false positives).** A guardrail whose signature matches
  legitimate work blocks good changes. Before promoting a new guardrail to *enforcing*,
  dry-run its trigger against recent history; if it would have fired on many
  non-mistakes, it's too broad — narrow it or keep it advisory. (This mirrors thymic
  negative selection: reject the antibody that attacks healthy tissue.)

- **Confidence updated by self-agreement.** If you track each guardrail's reliability,
  update that score **only from real outcome events** — a human correction, a test
  result, a reverted commit — *never* from the model agreeing with itself. Letting
  "the agent thinks this rule is good" raise the rule's confidence is a closed loop
  that manufactures false certainty.

## Why it's more than a linter

A linter encodes rules a human wrote up front. This pattern encodes rules the system
*learned from its own failures*, with provenance (which incident produced it), a
reliability score grounded in outcomes, and a promotion path from advisory →
enforcing as evidence accumulates. It is a feedback loop, not a static ruleset — the
agent's mistake surface shrinks over time instead of staying flat.

## Minimal viable version

You don't need the full apparatus to get value. The 80/20:

1. A structured `guardrails.json` (id, pattern, trigger, severity, defense).
2. A pre-action hook that pattern-matches the highest-severity, mechanically-detectable
   triggers and blocks with the reason.
3. A capture step in your "I made a mistake" workflow that appends a new record.

Start with the handful of mistakes that actually keep happening. Enforce only the ones
you can detect without false positives. Grow from there.
