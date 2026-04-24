# agent-skills

Portable Claude-authored skills for the [Agent Skills](https://agentskills.io) open specification. Install via `gh skill install SpencerGoss/agent-skills/<skill-name>` in any Agent Skills-compatible host (Claude Code, Codex, Gemini CLI, Copilot CLI, Cursor).

## Skills

| Skill | What it does |
|---|---|
| **devil-advocate** | Adversarial review with 4 mandatory counterarguments (technical risk, better alternative, hidden assumption, harm scenario). Gates significant decisions behind structured opposition. |
| **refactor-session** | Structure-changing workflow — never mixes refactoring with behavior change. Rule-of-three before abstraction, one-change-per-commit, safety-net tests first. |
| **tdd-workflow** | Red/Green/Refactor cycle with a hard gate (no implementation before a failing test). Includes characterization-test path for legacy code. |

All three use pure prompt structure — no runtime dependencies beyond git + a test runner + a shell.

## Installation

```bash
# Install a single skill
gh skill install SpencerGoss/agent-skills/devil-advocate
gh skill install SpencerGoss/agent-skills/refactor-session
gh skill install SpencerGoss/agent-skills/tdd-workflow

# Inspect before installing
gh skill preview SpencerGoss/agent-skills/devil-advocate
```

Requires `gh` CLI v2.90.0 or later. The installer auto-detects your agent host and places the skill in the correct directory (`.claude/skills/`, `.gemini/skills/`, or `.agents/skills/` per vendor).

## License

MIT. See `LICENSE` for details.

## Provenance

Skills authored and maintained by [@SpencerGoss](https://github.com/SpencerGoss). Each SKILL.md carries a `metadata.author` field pointing back here.

## Contributing

Issues and PRs welcome. Keep in mind:
- Skills here are deliberately **prompt-only** — no scripts, no runtime deps beyond what's already listed
- Changes that add vendor-specific features should preserve the vendor-neutral baseline
- Each SKILL.md must pass `gh skill publish --dry-run` before merge

## Related

- [Agent Skills specification](https://agentskills.io/specification)
- [GitHub skill docs](https://docs.github.com/agent-skills)
- [Claude Code skills](https://code.claude.com/docs/en/skills)
