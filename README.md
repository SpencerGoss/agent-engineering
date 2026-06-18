# agent-engineering

**Agent-engineering patterns and portable, prompt-only skills for LLM coding agents.**

Two things live here:

1. **[Engineering patterns](#engineering-patterns)** — write-ups of the harder problems
   in building reliable agent systems: multi-agent orchestration, adversarial multi-LLM
   review, and self-correcting guardrails. Grounded in current research, distilled from
   building and operating a multi-agent development system.
2. **[Skills](#skills)** — 11 installable [Agent Skills](https://agentskills.io)
   (`SKILL.md`) that any compatible host can drop in. **Prompt-only — no runtime
   dependencies, no scripts, no lock-in.** They encode disciplined engineering workflows
   (review, debug, spec, security, performance, decision-making) as structured agent prompts.

Everything is vendor-neutral: it runs across Claude Code, Codex, Gemini CLI, Copilot CLI,
Cursor, and any Agent Skills–compatible host.

---

## Engineering patterns

The interesting part. These document how to build agent systems that stay *correct* at
scale — not "more agents," but the topologies and verification disciplines that stop
multi-agent setups from collapsing into confident, agreed-upon noise.

| Pattern | The core idea |
|---|---|
| **[Multi-Agent Orchestration](patterns/multi-agent-orchestration.md)** | Fan out work to parallel specialized workers, then verify with an **independent adversarial check — never a vote.** Includes the six coordination rules, role/domain composition, and bounding disciplines that prevent runaway loops. |
| **[Adversarial Multi-LLM Council](patterns/adversarial-council.md)** | Get a *genuinely* diverse second opinion from multiple models. The key move is **frame isolation** — a premise-audit lane that attacks the question itself, defeating the agreement-collapse that makes naive "LLM debate" worthless. |
| **[Learned Guardrails](patterns/learned-guardrails.md)** | An "immune system" that codifies an agent's past mistakes into guardrails and **enforces them mechanically** before the mistake recurs — with explicit defenses against false positives and self-reinforcing confidence. |

Each is written to be reusable in any agent stack, with references to the underlying
research (information-theoretic bounds on closed-system reasoning, debate value in
shared- vs. compositional-knowledge regimes, graph-topology coordination).

---

## Skills

Installable `SKILL.md` files. Each carries explicit trigger conditions, hard rules, a
structured method, and a "common traps" section — so the agent follows a real workflow
instead of improvising.

| Skill | What it does |
|---|---|
| **[judge-panel](judge-panel/SKILL.md)** | Runs N independent attempts at a problem through a panel of independent judges, then synthesizes the winner — a tournament for wide-solution-space tasks where one-shot-iterated locks in a mediocre first idea. |
| **[devil-advocate](devil-advocate/SKILL.md)** | Adversarial review with four mandatory counterarguments (technical risk, better alternative, hidden assumption, harm scenario). Gates significant decisions behind structured opposition. |
| **[code-review-session](code-review-session/SKILL.md)** | Severity-ranked review of a **diff** (not the whole codebase), with a CRITICAL→LOW checklist and an adversarial "try to break it" pass. |
| **[debug-session](debug-session/SKILL.md)** | Systematic diagnosis — reproduce, hypothesize, isolate, fix — instead of guess-and-check. Never assumes the cause before evidence. |
| **[security-audit](security-audit/SKILL.md)** | OWASP-grounded review for secrets, injection, auth, and input-validation flaws at trust boundaries. |
| **[pre-mortem](pre-mortem/SKILL.md)** | Fragility analysis *before* deploy — "assume this failed in production; what broke?" — ranked by likelihood × blast radius. |
| **[performance-tuning](performance-tuning/SKILL.md)** | Measure-first optimization: profile, find the real bottleneck, fix, re-measure. No guessing, no premature optimization. |
| **[spec-driven-dev](spec-driven-dev/SKILL.md)** | A planning gate — no implementation until a spec (goal, scope, approach, rejected alternatives) exists. |
| **[refactor-session](refactor-session/SKILL.md)** | Structure-only changes that never mix with behavior change. Rule-of-three before abstraction, one change per commit, safety-net tests first. |
| **[tdd-workflow](tdd-workflow/SKILL.md)** | Red/Green/Refactor with a hard gate (no implementation before a failing test), plus a characterization-test path for legacy code. |
| **[decision-log](decision-log/SKILL.md)** | Lightweight architecture decision records — capture the choice, the alternatives rejected, and *why*, so future readers don't relitigate it. |

A recurring theme across the skills: **structured disconfirmation** (actively trying to
break your own work), **scope discipline** (do one thing, refuse scope creep), and
**explicit stop conditions**.

---

## Installation

```bash
# Install a single skill into any Agent Skills-compatible host
gh skill install SpencerGoss/agent-engineering/judge-panel
gh skill install SpencerGoss/agent-engineering/code-review-session
gh skill install SpencerGoss/agent-engineering/debug-session
# ...etc — see the table above for all 11

# Inspect before installing
gh skill preview SpencerGoss/agent-engineering/judge-panel
```

Requires `gh` CLI v2.90.0 or later. The installer auto-detects your agent host and
places the skill in the correct directory (`.claude/skills/`, `.gemini/skills/`, or
`.agents/skills/` per vendor). The patterns are plain Markdown — read them directly.

---

## Design principles

- **Prompt-only.** No runtime dependencies beyond git + a shell + a test runner. A skill
  is a *prompt structure*, not a program — portable, auditable, and host-agnostic.
- **Trigger-conditions-first.** Each skill's `description` lists when it fires, not what
  it does — so the host routes to it correctly instead of summarizing and skipping it.
- **Hard rules at the top.** The load-bearing non-negotiables sit where the agent reads
  them first.
- **Adversarial by default.** Review, verify, and break-it passes are built in — the
  cheapest place to catch a mistake is before it ships.

## License

MIT. See [`LICENSE`](LICENSE).

## Provenance

Authored and maintained by [@SpencerGoss](https://github.com/SpencerGoss). Each
`SKILL.md` carries a `metadata.author` field. Contributions welcome — keep skills
prompt-only and preserve the vendor-neutral baseline.

## Related

- [Agent Skills specification](https://agentskills.io/specification)
- [GitHub skill docs](https://docs.github.com/agent-skills)
- [Claude Code skills](https://code.claude.com/docs/en/skills)
