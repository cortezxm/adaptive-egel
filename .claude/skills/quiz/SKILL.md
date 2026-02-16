---
name: quiz
description: "Use this skill when the user runs /quiz, asks for a quiz, wants to be tested, or asks for practice questions on a topic."
---

# /quiz — Dynamic Assessment & Roadmap Update

## Purpose

Generate 5 EGEL-format questions for the current topic, evaluate answers conversationally, update scores with weighted average, reorder the roadmap, update `psyche.json` (self-efficacy, confidence calibration, strategy effectiveness), and detect burnout patterns.

## MANDATORY Actions

| # | Action |
|---|--------|
| **G1** | Respond to the user always in Spanish |
| **G2** | Read required state files before generating any content |
| **G3** | Append to `progress/sessions.jsonl` at the end of the session |
| **G4** | Atomic JSON writes: read complete → modify in memory → write complete |
| **G5** | If a state file is missing → create defaults and continue (never crash) |
| **Q1** | Generate exactly 5 questions with variety of EGEL question types |
| **Q2** | Include the 3 EGEL question types: interrogatorio directo, completamiento, relación |
| **Q3** | Use weighted average to update score: `new_score = old_score * 0.35 + quiz_score * 0.65` (never overwrite) |
| **Q4** | Update `times_failed_consecutively` correctly |
| **Q5** | Reorder `roadmap_order` after EACH quiz using the priority formula |
| **Q6** | Do NOT read the topic .md file to generate questions (use internal knowledge only) |
| **Q7** | Feedback per answer: acknowledge correct + gentle correction + key insight |
| **Q8** | If `times_failed_consecutively >= 3`: add note to `progress.json.notes[]` |
| **Q9** | Update `psyche.json` at session close (self-efficacy, calibration, strategy effectiveness) |

## Flow

### 1. Read State [G2, G5]

Read:
1. `progress/progress.json`
2. `user_profile/psyche.json`
3. `knowledge_base/index_map.json` (for topic metadata only, NOT the .md files)

If `psyche.json` has `status: "bootstrap-default"`, tell user: "Aún no has completado tu calibración. Ejecuta `/onboard` primero."

### 2. Determine Topic

Use `roadmap_order[0]` from `progress.json` as the quiz topic, unless the user specifies a different topic.

Validate that the topic exists in `index_map.json`.

### 3. Anxiety-Aware Framing

Check `psyche.json.anxiety_profile.trait_exam_anxiety`:

- If `trait_exam_anxiety > 0.7`:
  - **Never call it "quiz" or "examen"** — use **"práctica"** throughout
  - Present the first 3 questions and after those 3 ask: "¿Quieres continuar con 2 preguntas más?" before presenting the remaining 2
  - Open with: "Vamos a hacer una práctica rápida para ver cómo va [tema]."
- If `trait_exam_anxiety <= 0.7`:
  - Standard framing: "Quiz de [tema]"

### 4. Predicted Score (Metacognition Check)

Check `psyche.json.metacognition.self_monitoring`:

- If `self_monitoring < 0.4`:
  - Before presenting questions, ask: "¿Qué porcentaje crees que vas a sacar en esta práctica? (un número del 0 al 100)"
  - Store the response in `progress.json.topics[topic].predicted_score`
  - This will be used for calibration comparison after scoring

### 5. Calibrate Difficulty

Based on the topic's current state in `progress.json`:

| Condition | Difficulty Level |
|-----------|-----------------|
| `attempts == 0` | Basic (fundamental concepts) |
| `score < 70` | Basic + Intermediate |
| `score 70-84` | Satisfactorio (applied concepts) |
| `score >= 85` | Sobresaliente (analysis, edge cases, complex scenarios) |

**Overconfidence adjustment:** If `confidence_calibration.pattern == "overconfident"`:
- Present harder questions one level above the standard difficulty
- Include at least one edge case or counterexample question

### 6. Generate Questions [Q1, Q2, Q6, G1]

Generate exactly 5 questions **in Spanish** following EGEL format. **Do NOT read the topic .md file** — generate from internal knowledge.

**Required mix of question types:**
1. **Interrogatorio directo** (2-3 questions): Direct knowledge question with 4 options (A-D)
2. **Completamiento** (1-2 questions): Complete a statement, sequence, or fill-in-the-blank with 4 options
3. **Relación de elementos** (1 question): Match concepts with definitions, examples, or properties

Present all 5 questions at once (or first 3 if high-anxiety framing applies). Wait for user to answer.

### 7. Evaluate Answers [Q7]

For each answer, evaluate on a scale of 0.0 to 1.0:
- **1.0**: Fully correct
- **0.7-0.9**: Partially correct or correct reasoning with minor error
- **0.3-0.6**: Shows some understanding but significant gaps
- **0.0-0.2**: Incorrect

Calculate: `quiz_score = (sum of 5 scores / 5) × 100`

**Feedback per answer [Q7]:** Apply `psyche.json.feedback_profile`:
- `directness` → controls correction style (see /study Section 6 for details)
- `emotional_support` → controls validation amount
- `detail_level` → controls explanation depth
- If correct: Brief acknowledgment calibrated to `celebration_preference` + one additional insight
- If partially correct: Acknowledge what's right, gently explain the gap
- If incorrect: No shame. Explain the correct answer with a key insight. Use `zpd_level` to calibrate depth.

**Calibration reveal (if predicted_score was captured):**
After scoring all answers, compare:
- If `quiz_score >= predicted_score - 10 AND <= predicted_score + 10`: "Acertaste bastante bien en tu estimación — eso muestra buena autoconciencia."
- If `quiz_score > predicted_score + 10`: "Sacaste más de lo que esperabas — ¡buena señal!" → hint that they may be underestimating themselves
- If `quiz_score < predicted_score - 10`: "Sacaste menos de lo que estimabas. Vale la pena revisar los subtemas donde fallaste." → calibration gap signal

### 8. Update Score [Q3, G4]

```
If attempts == 0:
  new_score = quiz_score  (first attempt, no average needed)
Else:
  new_score = old_score * 0.35 + quiz_score * 0.65
```

Round to 1 decimal place.

### 9. Update Topic State [Q4]

Update the topic in `progress.json`:

```json
{
  "score": "<new_score>",
  "attempts": "<previous + 1>",
  "last_quiz_date": "<YYYY-MM-DD>",
  "status": "<see below>",
  "times_failed_consecutively": "<see below>",
  "egel_area": "<unchanged>",
  "predicted_score": "<from step 4 or null>"
}
```

**Status mapping:**
| Score | Status |
|-------|--------|
| >= 85 | `"dominado"` |
| 70-84 | `"en_progreso"` |
| 60-69 | `"requiere_repaso"` |
| < 60 | `"en_dificultad"` |

**times_failed_consecutively [Q4]:**
- If `quiz_score >= 70`: reset to `0`
- If `quiz_score < 70`: increment by `1`

### 10. Burnout Detection [Q8]

If `times_failed_consecutively >= 3` after this update:
- Add a note to `progress.json.notes[]`:
  ```json
  {"date": "<YYYY-MM-DD>", "type": "burnout-signal", "topic": "<topic-id>", "consecutive_failures": "<N>", "action": "pedagogy-switch-recommended"}
  ```
- Adapt the closing message: suggest reviewing with `/study` using a different approach, or suggest taking a break

### 11. Reorder Roadmap [Q5]

Apply the priority formula to ALL topics and regenerate `roadmap_order`:

```
score < 70 AND attempts > 0 → priority 1
times_failed_consecutively >= 3 → priority 2
score < 85 AND attempts > 0 → priority 3
score >= 85 → priority 4
attempts == 0 → priority 5
```

Respect prerequisites: skip topic if any prerequisite has `score < 60`.
Within same priority, maintain original order.

### 12. Update psyche.json [Q9, G4]

Read `user_profile/psyche.json`, apply the following updates, then write atomically.

**Damping rule:** No numeric field changes by more than ±0.1 per session to prevent noise-driven swings.

#### Self-Efficacy Update

```
area = topic.egel_area  (e.g., "1_algoritmia")

if quiz_score >= 70:
  self_efficacy.by_area[area] = min(1.0, self_efficacy.by_area[area] + 0.05)
if quiz_score < 50:
  self_efficacy.by_area[area] = max(0.0, self_efficacy.by_area[area] - 0.03)

self_efficacy.global = mean(all by_area values)
```

#### Confidence Calibration Update

```
area_key = "1_algoritmia" | "2_software_base" | etc. (matches topic.egel_area)

Update confidence_calibration.by_area[area_key].actual:
  if actual is null:
    actual = quiz_score / 100
  else:
    actual = actual * 0.5 + (quiz_score / 100) * 0.5  (rolling average)

Recompute calibration_gap:
  areas_with_data = [a for a in by_area if a.predicted != null AND a.actual != null]
  if len(areas_with_data) > 0:
    calibration_gap = mean(a.predicted - a.actual for a in areas_with_data)

Derive pattern:
  if calibration_gap > 0.15:  pattern = "overconfident"
  elif calibration_gap < -0.15: pattern = "underconfident"
  elif calibration_gap != null: pattern = "well_calibrated"
```

**Overconfidence response:** If `pattern` just became `"overconfident"`, at end of quiz message: "Tus predicciones tienden a ser más altas que tus resultados reales. En los próximos quizzes, pondré preguntas de mayor dificultad para ayudarte a calibrar mejor."

**Underconfidence response:** If `pattern` just became `"underconfident"`: After revealing score, say: "Sacaste X%, que es mayor de lo que esperabas. Eres más capaz de lo que crees."

#### Scaffolding Evolution

Based on `times_failed_consecutively` (after update in step 9):

```
if times_failed_consecutively >= 2:
  if zpd_level == "independent": move to "guided"
  elif zpd_level == "guided": move to "structured"
  elif zpd_level == "structured": move to "modeled"
  (never go below "modeled")

if quiz_score >= 85 AND attempts >= 2:
  if zpd_level == "modeled": move to "guided"
  elif zpd_level == "guided": move to "independent"
  (never go above "independent")
```

#### Strategy Effectiveness Tracking

Read `sessions.jsonl` to find strategies used in the most recent `/study` session for this topic. Then:

```
For each strategy in strategies_used (from last /study session log):
  if strategy not in strategy_effectiveness:
    strategy_effectiveness[strategy] = {"uses": 0, "followed_by_improvement": 0}

  strategy_effectiveness[strategy].uses += 1

  if new_score > old_score:
    strategy_effectiveness[strategy].followed_by_improvement += 1
```

After update, check for low-effectiveness strategies:

```
For each strategy in strategy_effectiveness:
  if uses >= 3:
    effectiveness = followed_by_improvement / uses
    if effectiveness < 0.3:
      flag: add note to progress.json.notes: {"type": "low-strategy-effectiveness", "strategy": strategy, "effectiveness": effectiveness}
```

**Update `last_updated`** → today.

### 13. Adaptive Response [G1]

Based on `quiz_score`, respond in Spanish:

- **< 70%**: "Hay algunos subtemas que necesitan refuerzo." List weak areas. Suggest: "Te recomiendo volver a `/study` enfocándote en [weak subtopics]."
- **70-84%**: "¡Nivel Satisfactorio alcanzado! Buen trabajo." Note what could improve for Sobresaliente.
- **>= 85%**: "¡Nivel Sobresaliente! Has dominado [topic]." Celebrate aligned to `feedback_profile.celebration_preference`.

If `times_failed_consecutively >= 3`, override with supportive message and suggest alternative study approach based on `active_strategies.frustration`.

### 14. Update Global State & Log [G3, G4]

**Update progress.json:**
- `current_date` → today
- `stress_level`: if `times_failed_consecutively >= 2` → "medio"; if `>= 3` → "alto"; else keep or lower
- `total_sessions` → increment by 1
- `study_streak`: if `last_study_date` is today → keep; if yesterday → keep; else → reset to 1

**Log session [G3]:**
Append to `progress/sessions.jsonl`:
```json
{"date":"<YYYY-MM-DD>","skill":"/quiz","topic":"<topic-id>","details":"score:<quiz_score>","metrics":{"new_score":"<new_score>","attempts":"<N>","stress":"<level>","zpd_level":"<current>","self_efficacy_area":"<updated_value>"}}
```

### 15. Next Step

Suggest what to do next based on score:
- If mastered: "Puedes avanzar al siguiente tema con `/study`."
- If progressing: "Intenta `/study` para reforzar antes del siguiente quiz."
- If struggling: "Tómate un descanso y vuelve con `/study` cuando estés listo."

If high anxiety (`trait_exam_anxiety > 0.7`), always use "práctica" instead of "quiz" in recommendations.
