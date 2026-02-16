---
name: dashboard
description: "Use this skill when the user runs /dashboard, asks to see their progress, wants to check their streak, or asks how they're doing on their study journey."
---

# /dashboard — Progress Dashboard

## Purpose

Read-only progress dashboard. Show streaks, mastered topics, weak areas, and the next recommended topic — all calibrated to the user's psychological profile. Never modifies `progress.json`.

## MANDATORY Actions

| # | Action |
|---|--------|
| **G1** | Respond to the user always in Spanish |
| **G2** | Read required state files before generating any content |
| **G3** | Append to `progress/sessions.jsonl` at the end of the session |
| **G4** | Atomic JSON writes: read complete → modify in memory → write complete |
| **G5** | If a state file is missing → create defaults and continue (never crash) |
| **ST1** | Read `progress/progress.json` AND `user_profile/psyche.json` before generating any output |
| **ST2** | If `progress.json` is missing → tell user (in Spanish) to run `/onboard` first |
| **ST3** | NEVER modify `progress.json` — this skill is strictly read-only on progress data |
| **ST4** | Calibrate tone using `feedback_profile`, `anxiety_profile`, and `motivation` from `psyche.json` |
| **ST5** | Celebrate streaks: if `celebration_preference > 0.6` → enthusiastic; if `<= 0.6` → brief acknowledgment |
| **ST6** | High exam anxiety (`trait_exam_anxiety > 0.7`) → normalize struggle, avoid alarming framing for weak topics |
| **ST7** | Display weak topics supportively — never shaming. Frame as "áreas por reforzar", not "temas fallidos" |
| **ST8** | Look up topic titles from `knowledge_base/index_map.json` (never show raw IDs to the user) |
| **ST9** | Append one session log entry to `progress/sessions.jsonl` after displaying the dashboard |

## Flow

### 1. Read State [ST1, ST2, G2, G5]

Read in order:
1. `progress/progress.json`
2. `user_profile/psyche.json`
3. `knowledge_base/index_map.json`

**If `progress.json` is missing [ST2]:**
Display in Spanish:
> "Parece que aún no has configurado tu perfil. Ejecuta `/onboard` para comenzar tu viaje de estudio."
Stop — do not continue.

**If `psyche.json` has `status: "bootstrap-default"` or is missing:**
Continue with the dashboard but skip all psyche-based tone calibration (use neutral defaults).

### 2. Compute Dashboard Metrics

From `progress.json`, compute:

- **Streak:** `study_streak` (days studied consecutively)
- **Total sessions:** `total_sessions`
- **Mastered topics:** topics where `score >= 85`
- **Progressing topics:** topics where `70 <= score < 85`
- **Weak topics:** topics where `score < 70` AND `attempts > 0`
- **Untouched topics:** topics where `attempts == 0`
- **Next topic:** `roadmap_order[0]` — look up its title from `index_map.json`
- **Global score:** weighted average of scores across all attempted topics (weight equally unless egel_area weighting is specified)
- **Readiness estimate:** `ceil(count of topics with score < 85 and attempts > 0)` study+quiz cycles remaining (label as approximate)

For each topic in the topic lists, use `index_map.json` to resolve the human-readable title [ST8].

### 3. Calibrate Tone [ST4, ST5, ST6, ST7]

Before rendering, determine tone parameters from `psyche.json`:

| Field | Condition | Effect |
|-------|-----------|--------|
| `feedback_profile.celebration_preference` | > 0.6 | Enthusiastic celebrations for streaks and mastered topics |
| `feedback_profile.celebration_preference` | <= 0.6 | Brief, calm acknowledgment |
| `feedback_profile.emotional_support` | > 0.6 | Empathetic framing for weak areas ("Es normal tener áreas por reforzar") |
| `anxiety_profile.trait_exam_anxiety` | > 0.7 | Normalize: never frame weak topics as alarming; emphasize how much is already done |
| `motivation.orientation` | `"mastery"` | Emphasize growth and progress, not just score numbers |
| `motivation.orientation` | `"performance_avoidance"` | Avoid competitive framing; use "avances" not "puntajes" |
| `study_habits.egel_attempts` | > 0 | Acknowledge exam experience ("Con tu experiencia previa, ya sabes cómo funciona el examen") |

### 4. Render Dashboard [G1]

Display a structured dashboard in Spanish. Use markdown formatting for readability.

```
📊 Tu Panel de Progreso — [user_name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🔥 Racha actual: [N] día(s)

📈 Progreso por temas:
  ✅ Dominados (≥85%):      [list of titles with scores, or "Ninguno aún"]
  📚 En progreso (70–84%):  [list of titles with scores, or "Ninguno aún"]
  🔄 Por reforzar (<70%):   [list of titles with scores — framed supportively]
  ⬜ Sin explorar:          [list of titles, or "Ninguno — ¡completaste todos!"]

🎯 Siguiente recomendación: [Topic Title]
   [1-sentence explanation of why this topic is next]

📅 Sesiones totales: [N]
📊 Puntaje global (temas intentados): [X]%
⏱️  Ciclos estudio+quiz restantes aprox.: ~[N]
```

**Topic list format:** Show title + score percentage for attempted topics. For untouched topics, show title only.

**Score display:** Round to 1 decimal place. Show `[title]: [score]%`.

**Weak topics framing [ST7]:** Lead with positives — e.g., "Tienes [N] tema(s) por reforzar (¡ya los identificaste, eso es lo importante!)".

### 5. Adaptive Closing Message [G1]

Tailor the closing message based on overall profile:

**Well progressing (≥50% of attempted topics are mastered or progressing):**
- Reinforce momentum. Connect to their `motivation.orientation`.
- Example: "Tu constancia está dando frutos. Sigue con `/study` para mantener el ritmo."

**Early stage (<20% of topics attempted):**
- Normalize the beginning. Encourage first steps.
- Example: "Estás al principio del camino — el momento perfecto para construir bases sólidas. Ejecuta `/study` para comenzar con [Next Topic]."

**Struggling (3+ topics in "weak" with `times_failed_consecutively >= 2`):**
- Empathetic, never alarming. Suggest the `/study` + `/quiz` cycle.
- Example: "Algunos temas necesitan más práctica — eso es completamente normal. Te recomiendo un ciclo `/study` → `/quiz` enfocado en [weakest topic title]."

**Streak ≥ 7 days:**
- Always celebrate explicitly regardless of `celebration_preference` (consistent effort deserves recognition).

**No sessions yet (total_sessions == 0 or all attempts == 0):**
- Welcome framing: "Tu viaje está por comenzar. Ejecuta `/study` para la primera sesión con [Next Topic]."

### 6. Log Session [ST9, G3, G4]

Append one entry to `progress/sessions.jsonl`:

```json
{"date":"YYYY-MM-DD","skill":"/dashboard","topic":null,"details":"dashboard-viewed","metrics":{"streak":<N>,"topics_mastered":<N>,"topics_progressing":<N>,"topics_weak":<N>,"stress":"<progress.stress_level>"}}
```

Use today's date (`2026-02-15` or the current date at runtime). Values come from the metrics computed in Step 2.
