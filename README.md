# Adaptive-EGEL

An intelligent tutoring system for the **EGEL-COMPU** exam, powered by Claude Code / Open Code / Gemini CLI. Transforms static study guides into a personalized, adaptive learning experience driven by evidence-based cognitive science.

---

## What is it?

Adaptive-EGEL helps CS students in Mexico prepare for the EGEL-COMPU (General Exam for Bachelor's Degree in Computing) through:

- **Personalized study sessions** that adapt to your learning level, motivation, and anxiety profile
- **Dynamic roadmap** that prioritizes topics based on your scores, failures, and prerequisites
- **Socratic questioning** instead of passive reading — you discover concepts by answering guided questions
- **Evidence-based profiling** built on self-efficacy theory (Bandura), Self-Determination Theory (Deci & Ryan), Vygotsky's ZPD, and metacognitive monitoring

---

## Quick Start

**1. Start your profile (run once):**

```
/onboard
```

This asks ~25 questions across 5 blocks (context, metacognition, motivation, anxiety, cognitive preferences) and generates your personalized profile.

**2. Begin a study session:**

```
/study
```

The tutor selects the best topic for you, loads the material, and guides you with Socratic questions calibrated to your level.

**3. Validate what you learned:**

```
/quiz
```

Five EGEL-format questions on the topic. Your score updates your roadmap and evolves your profile.

**4. Check your progress:**

```
/status
```

Dashboard showing streaks, mastered topics, weak areas, and next recommendation.

---

## Skills Reference

| Skill      | When to use                                        |
| ---------- | -------------------------------------------------- |
| `/onboard` | First time setup — generates your learning profile |
| `/study`   | Daily study session on your next roadmap topic     |
| `/quiz`    | 5-question assessment after a study session        |
| `/status`  | View your progress dashboard and streak            |

---

## Architecture

### Three-Layer Design

```
Knowledge Layer      → knowledge_base/ (study guides + topic index)
User State Layer     → user_profile/ + progress/ (psyche.json, progress.json, sessions.jsonl)
Orchestration Layer  → .claude/skills/ (stateless Claude Code skills)
```

**Key principle:** Every skill is stateless and file-driven. There is no database, no backend — all state lives in human-readable JSON files that you can inspect and modify directly.

### File Structure

```
adaptive-egel/
├── .claude/skills/
│   ├── onboard/SKILL.md      # Initial psychological & knowledge assessment
│   ├── study/SKILL.md        # Daily adaptive study session
│   ├── quiz/SKILL.md         # Dynamic assessment + roadmap update
│   └── status/SKILL.md       # Progress dashboard
├── knowledge_base/
│   ├── index_map.json         # Topic metadata (prerequisites, difficulty, EGEL area)
│   └── guias/                 # Study guide .md files (one per topic)
├── user_profile/
│   └── psyche.json            # Your learning profile (evidence-based, v2)
├── progress/
│   ├── progress.json          # Scores, streak, roadmap order
│   └── sessions.jsonl         # Append-only session history
├── README.md
├── CLAUDE.md                  # Instructions for Claude Code
├── first-prd.md               # Product Requirements Document
└── first-plan.md              # Implementation plan
```

---

## The EGEL-COMPU Exam

The EGEL-COMPU covers 4 areas:

| Area                                | Topics                                                           |
| ----------------------------------- | ---------------------------------------------------------------- |
| **Área 1 — Algoritmia**             | Algorithms, data structures, discrete mathematics, logic         |
| **Área 2 — Software de Base**       | Computer architecture, OS, compilers, networks                   |
| **Área 3 — Software de Aplicación** | Software engineering, programming languages, databases, security |
| **Área 4 — Cómputo Inteligente**    | AI, data mining, distributed computing                           |

Each area has two performance levels: **Satisfactorio** (passing) and **Sobresaliente** (distinction). The tutor covers both.

---

## Your Learning Profile (`psyche.json`)

Your profile is built on validated cognitive science — not debunked "learning style" inventories.

### What it measures

| Dimension                  | Based on                                | How it's used                                                                      |
| -------------------------- | --------------------------------------- | ---------------------------------------------------------------------------------- |
| **Self-efficacy**          | Bandura (1997)                          | Low → more scaffolding + "this is achievable" framing                              |
| **Metacognition**          | Flavell / Schraw                        | Low self-monitoring → insert comprehension checks                                  |
| **Motivation**             | Self-Determination Theory (Deci & Ryan) | Autonomy need → offer choices; External regulation → connect to EGEL outcomes      |
| **Cognitive profile**      | Cognitive Load Theory                   | Load tolerance → chunk size; Focus duration → break suggestions                    |
| **Mindset**                | Dweck (2006, refined)                   | Helplessness pattern → micro-steps + celebrate; Growth belief → normalize struggle |
| **Anxiety profile**        | Exam anxiety research                   | High trait anxiety → use "práctica" not "quiz"; Reduce scope under stress          |
| **Scaffolding (ZPD)**      | Vygotsky                                | Moves from modeled → guided → independent as mastery grows                         |
| **Confidence calibration** | Metacognitive monitoring                | Overconfident → harder questions; Underconfident → reveal positive gap             |
| **Strategy effectiveness** | Learning analytics                      | Strategies with < 30% effectiveness after 3 uses are flagged for replacement       |

### How it evolves

Your profile is **not static** — it updates after every session:

- `/onboard` — sets initial values from your 25-question interview
- `/study` — updates `state_anxiety` and `zpd_level` based on session behavior
- `/quiz` — updates `self_efficacy`, `confidence_calibration`, `scaffolding.zpd_level`, and `strategy_effectiveness`

**Damping rule:** No numeric field changes by more than ±0.1 per session to prevent noise-driven swings.

---

## Roadmap Logic

The tutor automatically prioritizes topics using this formula:

```
score < 70% AND attempted  → priority 1  (needs urgent review)
3+ consecutive failures    → priority 2  (activate deep support)
score 70-84% AND attempted → priority 3  (progressing, needs refinement)
score >= 85%               → priority 4  (mastered)
never attempted            → priority 5  (respect original order)
```

Prerequisites are always respected: a topic won't appear in your roadmap until its prerequisites reach 60%.

Score updates use a **weighted average** to preserve accumulated progress:

```
new_score = old_score × 0.35 + quiz_score × 0.65
```

---

## Configuration

You can inspect and manually edit your state files at any time:

- `user_profile/psyche.json` — your learning profile (all 0.0–1.0 numeric fields, string enums)
- `progress/progress.json` — topic scores, roadmap order, session streak
- `progress/sessions.jsonl` — full history of every session (append-only)

If you want to **reset your profile**, delete or reset `user_profile/psyche.json` to `status: "bootstrap-default"` and run `/onboard` again.

---

## Design Decisions

**Why file-based state (no database)?**

- Human-readable and version-controllable
- User can inspect or modify directly
- Aligns with Claude Code's stateless, file-first philosophy

**Why stateless skills?**

- Each invocation is independent and reproducible
- No hidden state between sessions
- Easier to debug

**Why evidence-based profiling instead of VARK?**

- VARK learning styles have no empirical support for improving outcomes (Pashler et al., 2008; Rogowsky et al., 2015)
- Self-efficacy, SDT motivation, and ZPD scaffolding have decades of evidence and directly map to pedagogical adaptations

---

## Credits

- EGEL-COMPU exam structure: CENEVAL (Centro Nacional de Evaluación para la Educación Superior)
- Original inspiration: [juanQNav/Egel-computer-science](https://github.com/juanQNav/Egel-computer-science)
- Built with [Claude Code](https://claude.ai/code)
