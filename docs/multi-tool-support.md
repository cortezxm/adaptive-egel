# Multi-Tool Support

Adaptive-EGEL skills use the [Agent Skills](https://agentskills.io) open standard and are compatible with three AI coding tools.

## Skill Discovery Paths

| Tool | Workspace Skills Directory | Standard |
|---|---|---|
| **Claude Code** | `.claude/skills/<skill-name>/SKILL.md` | Claude Code skills |
| **Gemini CLI** | `.gemini/skills/<skill-name>/SKILL.md` | Agent Skills open standard |
| **OpenAI Codex** | `.agents/skills/<skill-name>/SKILL.md` | Agent Skills open standard |
| **Open Code** | `.opencode/skills/<skill-name>/SKILL.md` | Agent Skills open standard |

> **Open Code note:** Also auto-discovers from `.claude/skills/` and `.agents/skills/` natively — no extra config needed. Config file: `.opencode/opencode.json`.

## Single Source of Truth (Symlinks)

Canonical skill definitions live in `.claude/skills/`. The other directories contain **symlinks** that point back to `.claude/skills/`:

```
.gemini/skills/onboard  →  ../../.claude/skills/onboard
.agents/skills/onboard  →  ../../../.claude/skills/onboard
```

**When adding a new skill**, always create it under `.claude/skills/` first, then add symlinks:

```bash
# After creating .claude/skills/<new-skill>/SKILL.md
ln -sfn "../../.claude/skills/<new-skill>"   ".gemini/skills/<new-skill>"
ln -sfn "../../../.claude/skills/<new-skill>" ".agents/skills/<new-skill>"
ln -sfn "../../../.claude/skills/<new-skill>" ".opencode/skills/<new-skill>"
```

## Optional Codex Configuration (`agents/openai.yaml`)

Each skill has an `agents/openai.yaml` file at `.claude/skills/<skill>/agents/openai.yaml` that configures invocation policies and model recommendations for the Codex app. This file is picked up automatically by Codex via the symlink.

## Recommended Models per Tool

| Tool | Recommended Models | Notes |
|---|---|---|
| **Claude Code** | `claude-sonnet-4-5` / `claude-opus-4-6` | Sonnet for daily use; Opus for complex sessions |
| **Gemini CLI** | `gemini-2.0-flash` / `gemini-2.0-pro` | Flash for speed; Pro for deep reasoning |
| **OpenAI Codex** | `gpt-4o` / `o3` | GPT-4o default; o3 for complex quiz/study sessions |
| **Open Code** | `moonshot/kimi-k2` (primary) · `minimax/m2.1` (secondary) | Kimi K2 for tutoring sessions; MiniMax for lightweight tasks; theme: `zen` |

## SKILL.md Format (shared across all tools)

All tools use the same YAML frontmatter format:

```yaml
---
name: skill-name
description: When this skill should trigger and what it does.
---
[Markdown instructions follow...]
```

The `description` field is what each tool uses for skill discovery and implicit activation — keep it precise and trigger-oriented.
