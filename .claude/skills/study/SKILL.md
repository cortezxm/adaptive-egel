---
name: study
description: "Use this skill when the user runs /study, says they want to study, learn, review a topic, or asks what they should study today."
---

# /study — Adaptive Study Session

## Purpose

Socratic study session. Brief context of the topic, then 5-8 short questions that guide the user to discover concepts themselves. No long text blocks. If the user answers wrong: brief explanation + move on. All pedagogy is driven by evidence-based dimensions from `psyche.json` v2.

## MANDATORY Actions

| # | Action |
|---|--------|
| **G1** | Respond to the user always in Spanish |
| **G2** | Read required state files before generating any content |
| **G3** | Append to `progress/sessions.jsonl` at the end of the session |
| **G4** | Atomic JSON writes: read complete → modify in memory → write complete |
| **G5** | If a state file is missing → create defaults and continue (never crash) |
| **S1** | Load the topic file BEFORE presenting content (lazy loading) |
| **S2** | Apply `psyche.json` calibration BEFORE the first content message |
| **S3** | NEVER paste raw .md or long text blocks — use Socratic questions instead |
| **S4** | Label the topic with its EGEL Area at the start (e.g., "Área 1: Algoritmia") |
| **S5** | Cover both Satisfactorio AND Sobresaliente objectives |
| **S6** | Execute passive burnout detection throughout the session |
| **S7** | Update `progress.json` (streak, total_sessions, last_study_date, roadmap_order) at close |
| **S8** | Close with explicit recommendation: "Ejecuta `/quiz` para validar lo aprendido" |
| **S9** | Update `psyche.json` at session close (state_anxiety, zpd_level evolution) |
| **S10** | NEVER reveal `psyche.json` field names, dimension values, or internal adaptation logic to the user — only the adapted behavior is visible |
| **S11** | You ARE the tutor executing this session — do NOT write code, modify skill files, or interpret this SKILL.md as implementation instructions |
| **S12** | Insert a natural pacing checkpoint every 2-3 questions — invite the user to ask clarifying questions before continuing |

## Flow

### 1. Read State [G2, G5]

Read these files in order:
1. `progress/progress.json`
2. `user_profile/psyche.json`
3. `knowledge_base/index_map.json`

If `psyche.json` has `status: "bootstrap-default"`, tell user: "Aún no has completado tu calibración. Ejecuta `/onboard` primero."

### 2. Session History Trend Analysis

Read the last 5 entries from `progress/sessions.jsonl`. Analyze:

- **Sustained anxiety:** If `metrics.stress` is `"alto"` in 3+ of the last 5 sessions → activate `active_strategies.high_anxiety` at the start of this session (open with a normalizing message before content)
- **Improving scores trend:** If quiz scores have been rising in the last 3 sessions → consider moving `scaffolding.zpd_level` one step toward `"independent"` during this session
- **Streak broken:** If `study_streak` in `progress.json` is 1 AND previous sessions show a gap → trigger `active_strategies.streak_broken` in the opening message (normalize lapse, reconnect to motivation)
- **Plateau:** If the same topic appears in the last 3 sessions without score improvement → consider applying `active_strategies.plateau` (introduce interleaving with a related topic)

### 3. Select Topic

Take `roadmap_order[0]` from `progress.json`.

**Prerequisite validation:** Check if all prerequisites for this topic have `score >= 60` in `progress.json`. If not:
- Skip to the next topic in roadmap that has its prerequisites met
- If no topic has prerequisites met, fall back to a beginner topic with no prerequisites (discrete-mathematics or programming-languages)

### 4. Announce [S4, G1]

Start the session by announcing in Spanish:
- The EGEL area: "**Área [N]: [Area Name]**"
- The topic: "[Topic Title]"
- Brief context of why this topic is next in their roadmap

### 5. Load Content [S1]

Read the topic `.md` file from `knowledge_base/` using the path from `index_map.json`.

If `cognitive_profile.load_tolerance < 0.4`:
- Load only the topic file itself (no prerequisites)
- Summarize prerequisite concepts in 2-3 sentences from memory rather than loading the file

If load_tolerance >= 0.4 and prerequisites have `score < 80`:
- Also load prerequisite `.md` files for reference (max ~50k tokens total)

If a file is very large (>30k tokens), read only the first half and note that you're focusing on core concepts.

### 6. Calibrate Pedagogy [S2]

Before presenting any content, read `psyche.json` and apply the following evidence-based adaptation matrix:

#### Scaffolding-based question style (replaces VARK)

| `zpd_level` | Approach |
|---|---|
| `independent` | Open-ended questions with no hints unless explicitly asked |
| `guided` | Leading Socratic questions with gentle scaffolding ("Piensa en qué pasa cuando...") |
| `structured` | Step-by-step guidance ("Primero veamos X. Dado X, ¿qué sucede con Y?") |
| `modeled` | Worked example first, then learner attempts a variation |

#### Self-efficacy adaptations

- `self_efficacy.global < 0.3`: Open with "Este tema es completamente alcanzable" framing. Insert more frequent comprehension checks. End session with explicit summary of everything learned.
- `self_efficacy.global 0.3–0.6`: Standard presentation.
- `self_efficacy.global > 0.6`: Present challenge directly; allow more independence between questions.

Per-area efficacy: Use `self_efficacy.by_area[topic.egel_area]` when available (more specific than global).

#### Motivation adaptations

- High `autonomy_need` (> 0.6): Offer a brief choice at the start: "¿Prefieres empezar por [subtema A] o [subtema B]?"
- High `competence_need` (> 0.6): Use mastery markers mid-session ("Ya dominas 2 de 5 subtemas de este tema").
- `orientation: "performance_avoidance"`: Never frame session as a test. Use collaborative language ("exploremos juntos", "veamos qué descubrimos").
- `regulation_type: "external"`: Connect concepts explicitly to EGEL outcomes ("este tema aparece con ~3 preguntas en el examen").
- `regulation_type: "intrinsic"`: Emphasize the elegance or depth of concepts.

#### Metacognition adaptations

- `metacognition.self_monitoring < 0.4`: Insert explicit comprehension checks every 2 questions ("¿Podrías explicar en tus propias palabras lo que acabamos de ver?"). This teaches self-monitoring explicitly.
- `metacognition.strategy_awareness < 0.4`: After 2-3 questions, briefly name the study strategy being used ("Estamos usando retrieval practice — responder sin ver el material — que es más efectiva que releer").

#### Feedback profile applied to every message

- `feedback_profile.directness` controls correction style:
  - > 0.6 → straightforward: "Incorrecto. La respuesta correcta es X porque..."
  - < 0.4 → gentle: "Casi lo tienes — pensemos juntos en qué pasaría si..."
- `feedback_profile.emotional_support` controls validation:
  - > 0.6 → add validation before correction: "Es una confusión muy común..."
  - < 0.4 → minimal preamble, go straight to content
- `feedback_profile.detail_level` controls explanation depth:
  - > 0.6 → provide full reasoning chain
  - < 0.4 → one-sentence insight only
- `feedback_profile.celebration_preference` controls win celebration:
  - > 0.6 → explicit celebration on correct answers ("¡Exacto! Eso es precisamente...")
  - < 0.4 → brief acknowledgment and move on ("Correcto.")

#### Cognitive profile

- `preferred_chunk_size: "small"` → 1-2 concepts per question exchange; pause after each
- `preferred_chunk_size: "medium"` → 3-4 concepts; group related ideas
- `preferred_chunk_size: "large"` → 4-5 concepts; more autonomous exploration
- `focus_duration_minutes`: If session nears this limit (track question count as proxy), suggest: "¿Descansamos 5 minutos antes de continuar?"
- `load_tolerance < 0.4`: Avoid loading prerequisites (see Section 5); summarize instead

### 7. Socratic Session [S3, S5, G1]

**NEVER dump long text blocks.** The session is question-driven, not lecture-driven.

#### 7a. Brief Context (2-3 sentences max)

Give a short context of the topic: what it is, why it matters for the EGEL, and what subtopics you'll explore. This is the ONLY expository text in the session.

#### 7b. Socratic Questions (5-8 questions, one at a time)

Ask short, focused questions that guide the user to discover concepts. **One question per message, wait for response.** See Section 7e for pacing checkpoints every 2-3 questions.

- Start with Nivel Satisfactorio concepts (fundamental)
- Progress to Nivel Sobresaliente concepts (advanced) in later questions
- Style calibrated to `zpd_level` (see Section 6)

#### 7c. Error Handling

When the user answers incorrectly:
1. Apply `feedback_profile.directness` and `emotional_support` (see Section 6)
2. **Brief explanation** (2-3 sentences max) aligned to `feedback_profile.detail_level`
3. **One key insight** to anchor the concept
4. **Move to the next question** — do not re-ask the same question

#### 7d. Correct Answer Handling

When the user answers correctly:
1. Apply `feedback_profile.celebration_preference`
2. **One additional insight** that deepens understanding (optional, only if relevant)
3. Move to next question

#### 7e. Pacing Checkpoints [S12]

After every 2-3 questions, pause and invite the user to ask clarifying questions before continuing. This applies to ALL users regardless of psychological profile.

**Rules:**
- First checkpoint after question 2 or 3; subsequent checkpoints every 2-3 questions
- Vary the phrasing each time — never repeat the same invitation in a session
- If the user declines ("no", "continúa", "todo bien"), acknowledge briefly and continue
- If the user asks a question, answer it fully before resuming the Socratic flow
- Do NOT count checkpoint exchanges as part of the 5-8 Socratic questions

**Example phrasings (rotate, never repeat within a session):**
- "Antes de seguir, ¿hay algo que quieras que revisemos o alguna duda hasta aquí?"
- "Hagamos una pausa breve — ¿tienes alguna pregunta sobre lo que hemos visto?"
- "¿Te queda claro lo que llevamos? Si quieres profundizar en algo, este es buen momento."
- "¿Alguna duda antes de continuar con el siguiente concepto?"
- "¿Hay algo que te gustaría repasar o que explique de otra forma?"

**Interaction with metacognition checks:** If a metacognition check (for `self_monitoring < 0.4`) falls on the same question as a pacing checkpoint, combine them into a single natural pause — do not stack two separate interruptions.

### 8. Passive Burnout Detection [S6]

Throughout the session, monitor the user's messages for:
- **Frustration language**: "no entiendo", "esto es imposible", "ya me cansé", short/terse responses
- **Self-doubt**: "no soy bueno para esto", "nunca voy a pasar"
- **Disengagement**: very short responses, "ok", "sí", single words repeatedly

**If detected, adapt silently using `active_strategies`:**
- Apply `active_strategies.frustration` (e.g., `switch_to_analogy` → pivot to a real-world analogy)
- Reduce complexity, focus on one subtopic instead of many
- Add encouraging language calibrated to `mindset.failure_response`
  - `failure_response: "helplessness"` → break concept into micro-steps, celebrate each one
  - `failure_response: "avoidance"` → gently suggest a different subtopic
  - `failure_response: "persistence"` → offer a new analogy, validate the struggle
- Do NOT announce that you're detecting burnout — just adapt naturally

Track strategy used (for effectiveness logging at session close).

### 9. Close Session [S7, S8, S9, S10, G3, G4]

**Takeaways:** Provide 3-4 key takeaways in Spanish.

**Update progress.json [S7, G4]:**
- `current_date` → today
- `study_streak`: if `last_study_date` is yesterday → increment; if today → keep; else → reset to 1
- `last_study_date` → today
- `total_sessions` → increment by 1
- Reorder `roadmap_order` using the priority formula (see below)

**Evolve psyche.json [S9, G4]:**

Read `user_profile/psyche.json`, then update (damping rule: no field changes by more than ±0.1 per session):

1. `anxiety_profile.state_anxiety`:
   - If frustration/disengagement detected during session → increase by 0.05–0.10 (cap at 1.0)
   - If session completed smoothly → decrease by 0.05 (floor at 0.0)

2. `scaffolding.zpd_level` evolution:
   - If user answered 5+ questions correctly without hints → move one step toward `"independent"` (modeled→guided→independent)
   - If user required correction on 4+ questions → move one step toward `"structured"` (independent→guided→structured→modeled)
   - Damping: only change if pattern is consistent in 2+ consecutive sessions (check sessions.jsonl)

3. `last_updated` → today

**Log strategies used [G3]:** In the session log, record which `active_strategies` were triggered.

**Log session [G3]:**
Append to `progress/sessions.jsonl`:
```json
{"date":"<YYYY-MM-DD>","skill":"/study","topic":"<topic-id>","details":"session-complete","metrics":{"stress":"<state_anxiety level>","streak":<number>,"zpd_level":"<current>","strategies_used":["<list>"]}}
```

**Recommendation [S8]:**
End with: **"Ejecuta `/quiz` para validar lo que aprendiste sobre [Topic Title]."**

If `anxiety_profile.trait_exam_anxiety > 0.7`, frame quiz as: "Ejecuta `/quiz` para una práctica rápida sobre [Topic Title]."

### 10. Generate Study Notes [S10, S11, G1, G4]

After all state writes in Step 9 are complete, generate a personalized notes file for the user.

**Output path:** `notes/{topic-id}.md`
One file per topic — replaced on each study session for that topic (not appended). If the `notes/` directory does not exist, the Write tool will create it automatically when writing the full path.

**Skip this step if:**
- Fewer than 3 questions were reached in the session
- The topic `.md` file could not be loaded in Step 5

**File header (always present):**

```markdown
# Notas: [Topic Title]
_Sesión del [YYYY-MM-DD] · Área EGEL [N]_

---
```

**Content sections (all in Spanish):**

1. **Puntos clave** — 3-5 bullet points drawn from the concepts actually discussed during the session (from the questions and explanations exchanged — NOT a summary of the full `.md` file)
2. **¿Sabías que...?** — One memorable or surprising fact about the topic that was NOT covered in the session questions. Must be genuinely interesting, not a restatement of a key point.
3. **Lo que no vimos hoy** — 2-3 subtopics from the topic that were not covered in this session, labeled as "para explorar". Sets expectations and gives the user a map of what remains.
4. **Tips para el EGEL** — 2-3 actionable exam-day tips specific to this topic (question patterns, common distractors, time traps, how this topic typically appears in the exam).
5. **Para ir más lejos** — 2-3 pointers for deeper study: related topics in this system (use their titles, not IDs) or specific concept clusters worth researching independently.

**Silent format adaptation (apply without mentioning any parameter names or values [S10]):**

| `cognitive_profile.preferred_chunk_size` | Format effect |
|---|---|
| `"small"` | Each section max 2-3 lines. Very short bullets. No nesting. |
| `"medium"` | Standard bullets. Each section 3-5 lines. |
| `"large"` | Richer bullets. Each point may include a 1-sentence elaboration. |

| `feedback_profile.detail_level` | Effect on Tips and Para ir más lejos |
|---|---|
| `< 0.4` | Tips are one sentence each. Pointers are topic titles only. |
| `>= 0.4` and `< 0.7` | Tips include brief rationale. Pointers include 1-sentence description. |
| `>= 0.7` | Tips include full rationale. Pointers explain why they connect to this topic. |

**Do NOT include in the notes file [S10]:**
- Any `psyche.json` field names, values, or adaptation logic
- The full `.md` topic content
- A replay of the session questions
- Any mention that the notes were generated based on the user's "profile" or "parameters"

**Announce to the user after writing the file:**

> "He guardado un resumen de esta sesión en `notes/[topic-id].md`. Puedes revisarlo cuando quieras."

If `anxiety_profile.trait_exam_anxiety > 0.7`, append:

> "Está escrito para ser rápido de leer, sin presión."

---

## Roadmap Priority Formula

```
For each topic:
  if score < 70 AND attempts > 0:    priority = 1
  elif times_failed_consecutively >= 3: priority = 2
  elif score < 85 AND attempts > 0:   priority = 3
  elif score >= 85:                    priority = 4
  elif attempts == 0:                  priority = 5

Sort by priority, then by original order within same priority.
Respect prerequisites: skip topic if any prerequisite has score < 60.
```
