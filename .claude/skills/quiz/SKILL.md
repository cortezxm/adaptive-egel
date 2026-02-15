---
name: quiz
description: "Use this skill when the user runs /quiz, asks for a quiz, wants to be tested, or asks for practice questions on a topic."
---

# /quiz — Dynamic Assessment & Roadmap Update

## Purpose

Generate 5 EGEL-format questions for the current topic, evaluate answers conversationally, update scores with weighted average, reorder the roadmap, and detect burnout patterns.

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

### 3. Calibrate Difficulty

Based on the topic's current state in `progress.json`:

| Condition | Difficulty Level |
|-----------|-----------------|
| `attempts == 0` | Basic (fundamental concepts) |
| `score < 70` | Basic + Intermediate |
| `score 70-84` | Satisfactorio (applied concepts) |
| `score >= 85` | Sobresaliente (analysis, edge cases, complex scenarios) |

### 4. Generate 5 Questions [Q1, Q2, Q6, G1]

Generate exactly 5 questions **in Spanish** following EGEL format. **Do NOT read the topic .md file** — generate from internal knowledge.

**Required mix of question types:**
1. **Interrogatorio directo** (2-3 questions): Direct knowledge question with 4 options (A-D)
2. **Completamiento** (1-2 questions): Complete a statement, sequence, or fill-in-the-blank with 4 options
3. **Relación de elementos** (1 question): Match concepts with definitions, examples, or properties

Present all 5 questions at once. Wait for user to answer all of them.

### 5. Evaluate Answers [Q7]

For each answer, evaluate on a scale of 0.0 to 1.0:
- **1.0**: Fully correct
- **0.7-0.9**: Partially correct or correct reasoning with minor error
- **0.3-0.6**: Shows some understanding but significant gaps
- **0.0-0.2**: Incorrect

Calculate: `quiz_score = (sum of 5 scores / 5) × 100`

**Feedback per answer [Q7]:** Provide feedback calibrated to `psyche.json`:
- If correct: Brief acknowledgment + one additional insight
- If partially correct: Acknowledge what's right, gently explain the gap
- If incorrect: No shame. Explain the correct answer with a key insight. Use the user's preferred learning style (example, analogy, etc.)

### 6. Update Score [Q3, G4]

```
If attempts == 0:
  new_score = quiz_score  (first attempt, no average needed)
Else:
  new_score = old_score * 0.35 + quiz_score * 0.65
```

Round to 1 decimal place.

### 7. Update Topic State [Q4]

Update the topic in `progress.json`:

```json
{
  "score": <new_score>,
  "attempts": <previous + 1>,
  "last_quiz_date": "<YYYY-MM-DD>",
  "status": "<see below>",
  "times_failed_consecutively": <see below>,
  "egel_area": <unchanged>
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

### 8. Burnout Detection [Q8]

If `times_failed_consecutively >= 3` after this update:
- Add a note to `progress.json.notes[]`:
  ```json
  {"date": "<YYYY-MM-DD>", "type": "burnout-signal", "topic": "<topic-id>", "consecutive_failures": <N>, "action": "pedagogy-switch-recommended"}
  ```
- Adapt the closing message: suggest reviewing with `/study` using a different approach, or suggest taking a break

### 9. Reorder Roadmap [Q5]

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

### 10. Adaptive Response [G1]

Based on `quiz_score`, respond in Spanish:

- **< 70%**: "Hay algunos subtemas que necesitan refuerzo." List weak areas. Suggest: "Te recomiendo volver a `/study` enfocándote en [weak subtopics]."
- **70-84%**: "¡Nivel Satisfactorio alcanzado! Buen trabajo." Note what could improve for Sobresaliente.
- **>= 85%**: "¡Nivel Sobresaliente! Has dominado [topic]." Celebrate aligned to `psyche.json` motivator.

If `times_failed_consecutively >= 3`, override with supportive message and suggest alternative study approach.

### 11. Update Global State & Log [G3, G4]

**Update progress.json:**
- `current_date` → today
- `stress_level`: if `times_failed_consecutively >= 2` → "medio"; if `>= 3` → "alto"; else keep or lower
- `total_sessions` → increment by 1
- `study_streak`: if `last_study_date` is today → keep; if yesterday → keep; else → reset to 1

**Log session [G3]:**
Append to `progress/sessions.jsonl`:
```json
{"date":"<YYYY-MM-DD>","skill":"/quiz","topic":"<topic-id>","details":"score:<quiz_score>","metrics":{"new_score":<new_score>,"attempts":<N>,"stress":"<level>"}}
```

### 12. Next Step

Suggest what to do next based on score:
- If mastered: "Puedes avanzar al siguiente tema con `/study`."
- If progressing: "Intenta `/study` para reforzar antes del siguiente quiz."
- If struggling: "Tómate un descanso y vuelve con `/study` cuando estés listo."
