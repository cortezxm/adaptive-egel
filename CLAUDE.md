# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**Adaptive-EGEL** is an intelligent tutoring system for the EGEL-COMPU exam. It transforms static study guides into an adaptive, personalized learning experience powered by Claude Code skills and LLMs.

**Key Principle:** Stateless skills that read/write JSON and Markdown files. Each skill invocation is independent; all state persists in the file system (no database, no external API calls beyond Claude).

---

## Architecture

### Three-Layer Design

1. **Knowledge Layer** (`knowledge_base/`)
   - All study guides and pedagogical content
   - `index_map.json`: Topic index with prerequisites, difficulty, estimated time
   - Lazy loading: Only load required topics per session

2. **User State Layer** (`user_profile/` + `progress/`)
   - `psyche.json`: Evidence-based psychological profile (self-efficacy, metacognition, motivation, anxiety, scaffolding)
   - `progress.json`: Scores, study streak, dynamic roadmap, stress level, predicted_score per topic
   - `sessions.jsonl`: Timestamped session history with strategies_used

3. **Orchestration Layer** (Claude Code Skills)
   - 6 core skills: `/onboard`, `/study`, `/quiz`, `/status`, `/roadmap`, `/burnout-check`
   - Each skill is stateless and file-driven
   - No persistent memory between invocations

### File Structure

```
adaptive-egel/
├── .claude/skills/
│   ├── onboard/          # Skill: Initial psychological + knowledge assessment
│   ├── study/            # Skill: Daily study session with adaptive content
│   ├── quiz/             # Skill: Dynamic assessment + roadmap update
│   ├── status/           # Skill: Progress dashboard
│   └── roadmap/          # Skill: Explain topic prioritization
├── knowledge_base/
│   ├── index_map.json    # Topic metadata index
│   └── guias/            # Study guide .md files
├── user_profile/
│   └── psyche.json       # v2 evidence-based profile (written by /onboard, /study, /quiz)
├── progress/
│   ├── progress.json     # User scores, streak, roadmap order
│   └── sessions.jsonl      # Session history
├── first-prd.md          # Product requirements document
└── first-plan.md         # Detailed implementation plan
```

---

## Core Data Models

### `psyche.json` (User Psychological Profile — v2)

**Written by `/onboard` (initial), then evolved by `/study` and `/quiz` after each session.**

Evidence-based dimensions replacing debunked VARK learning styles (Pashler et al., 2008):

```json
{
  "schema_version": 2,
  "status": "active",
  "user_name": "",
  "onboard_date": "",
  "last_updated": "",
  "self_efficacy": {
    "global": 0.5,
    "by_area": {
      "1_algoritmia": 0.5,
      "2_software_base": 0.5,
      "3_software_aplicacion": 0.5,
      "4_computo_inteligente": 0.5
    }
  },
  "metacognition": {
    "self_monitoring": 0.5,
    "calibration_accuracy": 0.5,
    "strategy_awareness": 0.5
  },
  "motivation": {
    "autonomy_need": 0.5,
    "competence_need": 0.5,
    "relatedness_need": 0.5,
    "orientation": "mastery",
    "regulation_type": "identified"
  },
  "cognitive_profile": {
    "load_tolerance": 0.5,
    "preferred_chunk_size": "medium",
    "focus_duration_minutes": 30,
    "daily_available_minutes": 60
  },
  "mindset": {
    "growth_belief": 0.5,
    "failure_response": "persistence",
    "attribution_style": "effort"
  },
  "anxiety_profile": {
    "trait_exam_anxiety": 0.5,
    "state_anxiety": 0.3,
    "failure_recovery_speed": "moderate",
    "anxiety_triggers": []
  },
  "scaffolding": {
    "zpd_level": "guided",
    "hint_preference": 0.5,
    "explanation_depth": "moderate"
  },
  "study_habits": {
    "spacing_awareness": 0.5,
    "retrieval_practice_awareness": 0.5,
    "self_regulation": 0.5,
    "egel_attempts": 0,
    "prior_formal_study": true
  },
  "feedback_profile": {
    "directness": 0.5,
    "emotional_support": 0.5,
    "detail_level": 0.5,
    "celebration_preference": 0.5
  },
  "confidence_calibration": {
    "by_area": {
      "1_algoritmia": { "predicted": null, "actual": null },
      "2_software_base": { "predicted": null, "actual": null },
      "3_software_aplicacion": { "predicted": null, "actual": null },
      "4_computo_inteligente": { "predicted": null, "actual": null }
    },
    "calibration_gap": null,
    "pattern": null
  },
  "active_strategies": {
    "default_pedagogy": "socratic_guided",
    "frustration": "switch_to_analogy",
    "plateau": "introduce_interleaving",
    "high_anxiety": "reduce_scope_and_celebrate",
    "overconfidence": "challenge_with_edge_cases",
    "underconfidence": "scaffold_and_celebrate_small_wins",
    "streak_broken": "normalize_and_reconnect"
  },
  "strategy_effectiveness": {}
}
```

**Research grounding for each dimension:**

| Field | Theory | Adaptation |
|---|---|---|
| `self_efficacy` | Bandura (1997) | Low efficacy → more scaffolding + framing ("achievable") |
| `metacognition` | Flavell/Schraw | Low self_monitoring → insert comprehension checks; teach strategies explicitly |
| `motivation` (SDT) | Deci & Ryan | autonomy_need → offer choices; regulation_type → connect to relevant goals |
| `cognitive_profile` | Cognitive Load Theory | load_tolerance → chunk size; focus_duration → break suggestions |
| `mindset` | Dweck (2006, refined) | failure_response helplessness → micro-steps; growth_belief < 0.4 → normalize struggle |
| `anxiety_profile` | Exam anxiety research | trait > 0.7 → use "práctica" not "quiz"; reduce scope under high state anxiety |
| `scaffolding.zpd_level` | Vygotsky ZPD | modeled→guided→independent as mastery grows |
| `confidence_calibration` | Metacognitive monitoring | Overconfident → harder questions; underconfident → reveal positive gap |
| `strategy_effectiveness` | Learning analytics | Replace strategies with < 30% effectiveness after 3+ uses |

**psyche.json is written by:** `/onboard` (initial), `/study` (state_anxiety + zpd_level), `/quiz` (self_efficacy + calibration + strategy_effectiveness + zpd_level).

**Damping rule:** No numeric field changes by more than ±0.1 per session.

### `index_map.json` (Knowledge Index)

Structure:

```json
{
  "topics": [
    {
      "id": "discrete-mathematics",
      "title": "Discrete Mathematics",
      "egel_area": 1,
      "file": "guias/discrete-mathematics.md",
      "difficulty": "beginner",
      "prerequisites": [],
      "estimated_time_minutes": 45,
      "subtopics": ["sets", "relations", "graph-theory"]
    }
  ]
}
```

### `progress.json` (User Progress)

Contains:

- `user_name`, `assessment_date`, `current_date`
- `study_streak`: Days studied consecutively
- `stress_level`: low/medium/high (updated by `/quiz` and `/study`)
- `topics`: Dict with score, attempts, last_quiz_date, status, times_failed_consecutively
- `roadmap_order`: Array of topic IDs ordered by priority

### `sessions.jsonl` (Session History)

Append-only JSONL (one JSON object per line). Each entry:

```json
{"date":"2026-02-15","skill":"/study","topic":"discrete-mathematics","details":"session-complete","metrics":{"stress":"low","duration_min":25}}
```

---

## The 6 Core Skills

### `/onboard`

**Purpose:** One-time initial setup. Ask ~25 questions across 5 evidence-based blocks, generate `psyche.json` v2 and initial `progress.json`.

**Key:** Only run once. Creates evidence-based baseline across 5 dimensions (self-efficacy, metacognition, motivation/SDT, anxiety, cognitive profile) and initial roadmap ordered by lowest self-efficacy areas.

### `/study`

**Purpose:** Daily study session with adaptive content and stress detection.

**Flow:**

1. Read `progress.json` + `psyche.json`
2. Optimize roadmap (prioritize topics with score < 70%, respect prerequisites)
3. Load main topic + prerequisites from `knowledge_base/` (lazy loading, max ~50-100k tokens)
4. Present material conversationally using scaffolding-based pedagogy (zpd_level, self_efficacy, motivation)
5. Passively detect stress and burnout; adapt silently using `active_strategies`
6. Evolve `psyche.json` at close (state_anxiety, zpd_level)
7. End with recommendation to run `/quiz`

**Token Budget:** ~20-30k per session, ~$0.05-0.10 cost

### `/quiz`

**Purpose:** Dynamic assessment + roadmap update.

**Important:** Do NOT read the topic `.md` file to generate questions. Generate from internal knowledge only. This prevents answer leakage and saves tokens.

**Flow:**

1. Generate 5 questions on the fly (difficulty adjusted to user level)
2. User answers conversationally
3. Evaluate each answer (nuanced, not binary)
4. Update `progress.json` (score, attempts, stress level, consecutive failures)
5. Update `psyche.json` (self-efficacy, confidence calibration, scaffolding level, strategy effectiveness)
6. Adaptive response based on score, calibration pattern, and anxiety level

**Token Budget:** ~5-15k per quiz, ~$0.02-0.05 cost

### `/status`

**Purpose:** Progress dashboard showing streaks, mastered topics, weak areas, next recommendation.

**Tone:** Celebratory if progressing, supportive if struggling. Use `psyche.json` to calibrate.

### `/roadmap`

**Purpose:** Explain why topics are prioritized the way they are. Show prerequisites.

### `/burnout-check`

**Integration:** NOT a separate skill. Built into `/study` and `/quiz`.

**Detection Heuristics:**

- 3+ consecutive failures in same topic
- Frustration/self-doubt language patterns
- Stress level trending upward over 3+ sessions
- Streak broken after 5+ days

**Automatic Adaptation:** Switch pedagogy (theory → analogies/games), suggest breaks, celebrate small wins. Update `psyche.json` with new strategy.

---

## Implementation Guidelines

### For All Skills

1. **Read files at start:** Always read relevant JSON/MD before generating content
2. **Write atomically:** Read → modify in memory → write complete file (never partial updates)
3. **Timestamp everything:** Add `last_updated` and timestamps to tracked fields
4. **Conversational tone:** Adapt to user's `psyche.json`. No robotic outputs.
5. **Error handling:** If a file is missing, create sensible defaults (don't crash)

### Knowledge Loading

- Use `index_map.json` to find the right `.md` file
- Respect lazy loading: load topic + prerequisites only
- If a `.md` is huge, summarize or chunk it
- Max ~50-100k tokens of knowledge content per session

### Pedagogy Adaptation (Evidence-Based)

- **Never use VARK-based branching** — the VARK learning styles model is not supported by research (Pashler et al., 2008)
- Use `scaffolding.zpd_level` instead of `learning_style` to choose question style (modeled / guided / structured / independent)
- Use `self_efficacy.by_area` + `self_efficacy.global` to calibrate framing and scaffolding amount
- Use `motivation.orientation` and `regulation_type` for motivational framing
- Use `feedback_profile` (directness, emotional_support, detail_level, celebration_preference) for every feedback message
- Use `cognitive_profile.preferred_chunk_size` and `focus_duration_minutes` for session pacing

### Stress & Burnout Detection

- Be proactive, not reactive. Adapt without asking permission.
- Log all burnout detection decisions in `progress.json` notes field
- Never shame user for struggling
- If 3+ consecutive failures detected, activate `active_strategies.frustration` — never keep same pedagogy
- High `state_anxiety` detected from language → apply `active_strategies.high_anxiety`
- Sustained high stress (3+ sessions) → trigger anxiety intervention at session start

### Roadmap Logic

```
For each topic:
  if score < 70 AND attempts > 0:
    priority = 1 (highest — needs review)
  elif times_failed_consecutively >= 3:
    priority = 2 (activate deep analysis / burnout detection)
  elif score < 85 AND attempts > 0:
    priority = 3 (progressing, needs refinement)
  elif score >= 85:
    priority = 4 (mastered)
  elif attempts == 0:
    priority = 5 (never attempted — respect original order)

Sort by priority → roadmap_order

Respect prerequisites: Don't suggest topic X if prerequisite Y has score < 60%
```

### Score Update Formula

When updating topic scores after a quiz:

```
new_score = old_score * 0.35 + quiz_score * 0.65
```

Never overwrite — always use weighted average to preserve accumulated progress.

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

## Key Design Decisions

**Why Stateless Skills + File-Based State:**

- Independent skill invocations (no persistent memory)
- All state in human-readable JSON/MD (version-controllable)
- Lazy loading enables token efficiency
- Easier to debug and understand system state

**Why No Database:**

- Simpler deployment and maintenance
- User can inspect/modify files directly
- File-based allows easy integration with version control
- Aligns with Claude Code's file-first philosophy

**Why Hybrid Roadmap Optimizer:**

- Simple logic (score < 70% → prioritize) handles most cases
- Deep analysis only when anomalies detected (3+ failures)
- Balances intelligence with token efficiency

---

## Development Workflow

1. **Setup:** Review `first-plan.md` and `first-prd.md` for full context
2. **Implement skills in order:** `/onboard` → `/study` → `/quiz` → `/status` → `/roadmap`
3. **Before implementing:** Ensure directories exist under `.claude/skills/`
4. **For each skill:** Create `SKILL.md` with skill definition and implementation notes
5. **Test:** Run full flow: `/onboard` → `/study` → `/quiz` → `/status`
6. **Monitor:** Check token usage and user engagement metrics

---

## Key References

- `first-prd.md`: Product Requirements Document with vision, objectives, and use cases
- `first-plan.md`: Detailed implementation plan with technical specifications
- Original inspiration: https://github.com/juanQNav/Egel-computer-science

---
