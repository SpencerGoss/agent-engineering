---
name: decision-log
description: >
  Use when choosing between named alternatives (libraries, tools, algorithms, schemas, approaches),
  when making an architectural decision that has lasting consequences, when the user asks "should we use
  X or Y", "which approach is better", or "why did we choose X", or when you pick one option over
  another for non-obvious reasons. Trigger whenever a choice is made that future work would need to
  understand. For progress logging of completed work (what was built, TODOs), use a project journal
  instead. This skill = decision rationale storage for lasting architectural choices.
license: MIT
compatibility: "Works across Claude Code, Codex, Gemini CLI, Copilot CLI, and any Agent Skills-compatible host. No runtime dependencies — pure prompt structure."
metadata:
  version: "1.0.0"
  author: SpencerGoss
---

## Hard Rules

- **Only log decisions future work would need to understand.** Trivially reversible preferences (formatting, naming taste, one-off config values) do not get logged. A constrained, lasting, architectural choice does.
- **Never log a decision without its alternatives.** The Alternatives Considered section is the most valuable part of the record. A decision with no alternatives is just a statement and provides no future value.
- **Rationale must name a specific tradeoff.** "It was better" is not rationale. Name the concrete reason this option won (e.g. lower operational overhead, no added dependency, matches existing skills).
- **A reversed decision must be marked `Superseded`, not deleted.** Add a new entry explaining the reversal and link the two. The history of why a choice changed is itself load-bearing.
- **The log only has value if it is read.** Consult it at the start of work and whenever a settled topic is re-opened — otherwise the same decisions get re-litigated from scratch.

---

# Decision Log

Prevents re-litigating past decisions every few work sessions. Writes architectural and design
decisions to a single `DECISIONS.md` file with rationale, the alternatives considered, and the
conditions that would trigger revisiting the choice. Without this, the same decisions get re-made
from scratch repeatedly, often inconsistently.

Canonical persistence: one markdown file, `DECISIONS.md`, at the project root, append-only, newest
entries at the bottom. No database, no external store — a plain file any future session or teammate
can read.

---

## Steps

### 1. Identify the Decision

- State what is being decided in one sentence: "Choosing X over Y for [purpose]."
- Is this a lasting decision that would affect future code or architecture? If no — skip this skill.
- Is it a constrained choice or just a preference? Preferences don't need logging; constrained choices do.

### 2. Check DECISIONS.md for an Existing Entry

```bash
[ -f DECISIONS.md ] && grep -i "[keyword from decision]" DECISIONS.md || echo "No existing entry"
```

If a related decision already exists, update it rather than creating a duplicate. Change the old
entry's `Status` to `Superseded` and link it to the new entry.

### 3. Write the Decision Entry

If `DECISIONS.md` does not exist, create it with this header first:

```markdown
# Project Decisions

Architectural and design decisions made in this project, with rationale and alternatives.
Consult this before re-opening settled questions.

---
```

Then append the new entry:

```markdown
## [Short Decision Title]
Date: [DATE]
Status: Active

### Decision
[One sentence: what was decided]

### Context
[1-2 sentences: why this decision was needed, what problem it solves]

### Alternatives Considered
- **[Option A]**: [why rejected or not chosen]
- **[Option B]**: [why rejected or not chosen]
- **[Chosen option]**: [why this was selected]

### Rationale
[2-3 sentences: the reasoning that led to this choice]

### Consequences
- What becomes easier: [...]
- What becomes harder: [...]
- What this locks us into: [...]

### Revisit If
[What circumstances would cause a re-evaluation of this decision]
```

### 4. Reference in the Project Journal (optional)

If the decision is significant, mention it in the project's build log / journal so progress tracking
points back at the rationale:

```
Decision logged: [title]
```

### 5. Confirm

Tell the user:

> "Logged to DECISIONS.md: **[title]**. Read DECISIONS.md to review all logged decisions."

---

## Skill Chain

| Stage | Skill |
|-------|-------|
| Before (evaluating options) | **spec-driven-dev** — planning may surface decisions to log |
| Stress-testing the choice first | **devil-advocate** — argue against the preferred option before committing |
| Wide solution space, no obvious winner | **judge-panel** — run competing attempts, then log the winner here |
| This skill | **decision-log** — write the decision to DECISIONS.md |
| After (session end) | commit DECISIONS.md alongside the rest of the session's changes |

---

## Trigger Conditions

- Choosing between named alternatives — libraries, tools, algorithms, schemas, data models, or approaches.
- Making an architectural or design decision with lasting consequences.
- The user asks "should we use X or Y", "which approach is better", or "why did we choose X".
- You pick one option over another for a non-obvious reason that a future session would otherwise re-question.

## Out of Scope

- NOT for logging daily progress or completed work — use a project journal / build log instead.
- NOT for capturing reusable debugging insights — capture those as a separate engineering/lessons note instead.
- NOT for planning or designing features from scratch — use **spec-driven-dev**.
- NEVER use this for trivially reversible preferences that don't affect architecture.

## Common Traps

- **Logging every tiny choice** — only log decisions future work would need to understand. Skip obvious or trivially reversible ones; the log loses signal if it fills with noise.
- **Logging without alternatives** — the Alternatives Considered section is the most valuable part. A decision without alternatives is just a statement. Never skip it.
- **Never reading DECISIONS.md** — the file has no value if it isn't consulted. Reference it at the start of work and whenever a topic from it is re-opened.
- **Stale "Active" status** — decisions that have been reversed must be updated to `Status: Superseded` with a new entry explaining the reversal.
- **Vague rationale** — "it was better" is not rationale. Name the specific tradeoff that made this option win (e.g. lower operational overhead, no additional dependency, matches existing team skills).
