# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**Adaptive-EGEL** is an intelligent tutoring system for the EGEL-COMPU exam. It transforms static study guides into an adaptive, personalized learning experience powered by Claude Code skills and LLMs.

**Key Principle:** Stateless skills that read/write JSON and Markdown files. Each skill invocation is independent; all state persists in the file system (no database, no external API calls beyond Claude).

---

## Architecture

### Three-Layer Design

1. **Knowledge Layer** (`knowledge_base/`): Study guides + `index_map.json` topic index. Lazy loading only.
2. **User State Layer** (`user_profile/` + `progress/`): `psyche.json` (psychological profile), `progress.json` (scores/roadmap), `sessions.jsonl` (history).
3. **Orchestration Layer** (Skills): 6 stateless, file-driven skills.

### File Structure

```
adaptive-egel/
├── .claude/skills/        # Skill definitions (onboard, study, quiz, dashboard, roadmap)
├── knowledge_base/
│   ├── index_map.json     # Topic metadata index
│   └── guias/             # Study guide .md files
├── user_profile/
│   └── psyche.json        # Evidence-based psychological profile
├── progress/
│   ├── progress.json      # Scores, streak, roadmap order
│   └── sessions.jsonl     # Session history (append-only JSONL)
└── docs/                  # Design decisions, dev workflow, multi-tool setup
```

---

## Core Data Models

### `psyche.json` (User Psychological Profile)

Evidence-based profile written by `/onboard` (initial), then evolved by `/study` and `/quiz` after each session. Replaces debunked VARK learning styles (Pashler et al., 2008).

**10 dimensions:** `self_efficacy` (global + by area), `metacognition` (monitoring, calibration, strategy awareness), `motivation` (SDT: autonomy, competence, relatedness), `cognitive_profile` (load tolerance, chunk size, focus duration), `mindset` (growth belief, failure response), `anxiety_profile` (trait/state anxiety, triggers), `scaffolding` (ZPD level, hint preference), `study_habits` (spacing, retrieval practice), `feedback_profile` (directness, emotional support, detail level), `confidence_calibration` (predicted vs actual per area).

Also contains `active_strategies` (pedagogy switches for frustration, plateau, anxiety, overconfidence) and `strategy_effectiveness` tracking.

**Damping rule:** No numeric field changes by more than ±0.1 per session.

Full schema: see `user_profile/psyche.json`.

### `index_map.json` (Knowledge Index)

Array of topic objects with: `id`, `title`, `egel_area` (1-4), `file` (path to guide), `difficulty`, `prerequisites`, `estimated_time_minutes`, `subtopics`.

### `progress.json` (User Progress)

- `study_streak`: Days studied consecutively
- `stress_level`: low/medium/high (updated by `/quiz` and `/study`)
- `topics`: Dict with score, attempts, last_quiz_date, status, times_failed_consecutively
- `roadmap_order`: Array of topic IDs ordered by priority

### `sessions.jsonl` (Session History)

Append-only JSONL. Each entry: `{"date", "skill", "topic", "details", "metrics": {"stress", "duration_min", "methodology", "comprehension_depth", "bridges_used"}}`.

---

## The 6 Core Skills

Each skill has detailed implementation instructions in its own `SKILL.md` file under `.claude/skills/<skill-name>/`.

- **`/onboard`**: One-time setup. ~25 questions across 5 evidence-based blocks → generates `psyche.json` and initial `progress.json`.
- **`/study`**: Adaptive study session (Socratic/Feynman/PBL). Reads psyche + progress, teaches topic with scaffolding, logs session.
- **`/quiz`**: 5-question assessment generated from internal knowledge (never read topic .md). Updates scores (weighted avg), psyche, and roadmap.
- **`/dashboard`**: Progress dashboard — streaks, mastered topics, weak areas, next recommendation.
- **`/roadmap`**: Explains topic prioritization logic and prerequisites.
- **`/burnout-check`**: Not a separate skill — built into `/study` and `/quiz`. Detects frustration, consecutive failures, rising stress.

---

## Shared Rules (All Skills)

1. **Read files at start:** Always read relevant JSON/MD before generating content.
2. **Write atomically:** Read → modify in memory → write complete file (never partial updates).
3. **Timestamp everything:** Add `last_updated` and timestamps to tracked fields.
4. **Conversational tone:** Adapt to user's `psyche.json`. No robotic outputs.
5. **Error handling:** If a file is missing, create sensible defaults (don't crash).
6. **Damping rule:** No `psyche.json` numeric field changes by more than ±0.1 per session.
7. **No VARK:** Never use VARK-based branching — use `scaffolding.zpd_level` instead.

---

## Conventions & Style

- **User-facing content:** Spanish (the EGEL exam is in Spanish; study guides are in Spanish)
- **System files:** English (CLAUDE.md, SKILL.md definitions, code comments)
- **Filenames:** snake_case (`index_map.json`, `psyche.json`, `sessions.jsonl`)
- **Skill names:** kebab-case (`/onboard`, `/study`, `/quiz`, etc.)
- **JSON:** Pretty-printed, 2-space indentation
- **Markdown:** Clear heading hierarchy, no unnecessary formatting
- **Error messages:** Supportive tone, actionable guidance, in Spanish
- **Code in skills:** Use Read/Edit/Write tools (not Bash) for file operations where possible

---

## References

See `docs/` directory for: design decisions, development workflow, multi-tool support (symlinks, models per tool, SKILL.md format), and key references (PRD, plan, inspiration repo).
