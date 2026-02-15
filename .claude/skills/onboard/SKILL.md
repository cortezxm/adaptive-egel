---
name: onboard
description: "Use this skill when the user runs /onboard, wants to start, set up the tutor, do the initial assessment, or begin their study journey for the first time."
---

# /onboard — Initial Psychological & Knowledge Assessment

## Purpose

One-time calibration session. Ask ~25 questions one at a time (dynamically adding more when weaknesses are detected), then IMMEDIATELY write `psyche.json` (v2) and initialize `progress.json`. No confirmation prompts — writing is mandatory and automatic.

## MANDATORY Actions

| # | Action |
|---|--------|
| **G1** | Respond to the user always in Spanish |
| **G2** | Read required state files before generating any content |
| **G3** | Append to `progress/sessions.jsonl` at the end of the session |
| **G4** | Atomic JSON writes: read complete → modify in memory → write complete |
| **G5** | If a state file is missing → create defaults and continue (never crash) |
| **O1** | Check idempotency: if `psyche.json` exists and `status` is not `bootstrap-default`, ask user if they want to recalibrate |
| **O2** | One question per message (never two together) |
| **O3** | Get user's name in the welcome message |
| **O4** | Write `psyche.json` IMMEDIATELY after all questions — NO confirmation prompt, writing is mandatory |
| **O5** | Initialize `progress.json` with all 15 topics from `index_map.json` |
| **O6** | Generate `roadmap_order` respecting prerequisites |
| **O7** | Dynamic deepening: if user reveals a weakness or low confidence in an area, add 2-3 follow-up questions to understand the gap better |
| **O8** | NEVER ask for confirmation before writing files — all writes are automatic and mandatory |
| **O9** | Always present options as a numbered list (1. 2. 3. …) — never inline with slashes or commas |
| **O10** | NEVER leave a question fully open-ended — every non-scale question MUST offer a numbered list of suggested answers AND include a final escape option (e.g., "5. Otra — cuéntame" or "5. Algo diferente — escribe lo tuyo"). Pure Likert scales (1–5 numeric ratings) are exempt because the scale itself is the complete answer space. This rule applies to ALL scenario, context, and follow-up questions. |

## Flow

### 1. Check Idempotency [O1]

Read `user_profile/psyche.json`. If it exists and `status` is NOT `bootstrap-default`:
- Tell the user they already have a profile
- Ask if they want to recalibrate (this will overwrite their profile)
- If they say no, suggest `/study` instead and end

### 2. Welcome & Name [O3, G1]

Greet the user warmly in Spanish. Ask for their name. Explain:
- "Vamos a hacer una calibración de ~25 preguntas para personalizar tu experiencia de aprendizaje"
- "Una pregunta a la vez, sin prisa — esto toma unos 20 minutos"

### 3. Interview — One Question Per Message [O2, O9, O10]

Ask questions in the 5 blocks below. **One question per message, wait for response before next.**

**IMPORTANT [O9]:** Every question MUST present options as a numbered list. Never inline (e.g. "a / b / c").

**IMPORTANT [O10]:** EVERY question — including scale questions, scenario questions, and follow-ups — MUST end with a numbered escape option like "5. Otra — cuéntame" or "4. Algo diferente — escribe lo tuyo". There are NO fully open-ended questions in this interview. If the user picks "Otra", ask them to elaborate (that follow-up can also offer common options + escape).

Example format:
> ¿Cuánto tiempo puedes concentrarte antes de necesitar un descanso?
> 1. 15 minutos
> 2. 30 minutos
> 3. 45 minutos
> 4. 1 hora o más
> 5. Algo diferente — cuéntame

---

#### Block A: Contexto y Autoeficacia (7 preguntas) [→ self_efficacy, study_habits]

**A1:** "¿Has presentado el EGEL antes?"
1. No, será mi primera vez
2. Sí, una vez — no lo pasé
3. Sí, una vez — lo pasé pero quiero mejorar mi nivel
4. Sí, más de una vez — sigo intentándolo
5. Algo diferente — cuéntame
- Map: `study_habits.egel_attempts` (1→0, 2-4→1+); note initial anxiety signal from options 2 and 4

**A2 (escala):** "En general, ¿qué tan capaz te sientes de aprender y dominar temas nuevos de Ciencias Computacionales cuando te lo propones?"
1. 1 — Muy pocas veces lo logro
2. 2
3. 3 — A veces sí, a veces no
4. 4
5. 5 — Generalmente lo logro si me esfuerzo
- Map: `self_efficacy.global` (score/5 as 0.0–1.0)

**A3:** "¿Qué tan seguro/a te sientes con el Área 1 — Algoritmia (algoritmos, estructuras de datos, matemáticas discretas, lógica)?"
1. 1 — Sin confianza
2. 2
3. 3 — Confianza media
4. 4
5. 5 — Muy confiado/a
- Map: `self_efficacy.by_area.1_algoritmia`

**A4:** "¿Y con el Área 2 — Software de Base (arquitectura, sistemas operativos, compiladores, redes)?" (same 1–5 scale)
- Map: `self_efficacy.by_area.2_software_base`

**A5:** "¿Y con el Área 3 — Software de Aplicación (ingeniería de software, lenguajes, bases de datos, seguridad)?" (same 1–5 scale)
- Map: `self_efficacy.by_area.3_software_aplicacion`

**A6:** "¿Y con el Área 4 — Cómputo Inteligente (IA, minería de datos, cómputo distribuido)?" (same 1–5 scale)
- Map: `self_efficacy.by_area.4_computo_inteligente`

**A7:** "¿Cómo has aprendido los temas de Ciencias Computacionales hasta ahora?"
1. Principalmente en la universidad — clases formales
2. Mitad universidad, mitad autodidacta (cursos online, libros, proyectos)
3. Casi todo autodidacta — cursos online, YouTube, libros
4. Aprendizaje en el trabajo o práctica profesional
5. Algo diferente — cuéntame
- Map: `study_habits.prior_formal_study` (1 or 2 → `true`; 3 or 4 → depends on context)

**Dynamic deepening [O7, O10]:** If any area score ≤ 2, add 2-3 follow-ups (always with numbered options + escape):

Follow-up 1: "¿Qué parte de [área] te resulta más difícil?"
1. Los conceptos teóricos y definiciones formales
2. Resolver ejercicios o problemas prácticos
3. Recordar los detalles cuando los necesito
4. Conectar los temas entre sí
5. Algo diferente — cuéntame

Follow-up 2: "¿Has tenido exposición formal a [área] o es algo que casi no has visto?"
1. Tuve una materia en la universidad pero no recuerdo mucho
2. Lo vi de forma autodidacta pero sin profundidad
3. Casi no lo he estudiado — es terreno nuevo para mí
4. Lo conozco de la práctica profesional pero no formalmente
5. Algo diferente — cuéntame

Follow-up 3 (optional): "¿Hay algún subtema dentro de [área] que sí domines o te genere más confianza?"
1. Sí — [el tutor debe dejar que el usuario escriba el subtema, o deducirlo del contexto]
2. No realmente, todo el área se me complica igual
3. No estoy seguro/a de cuáles son los subtemas
4. Algo diferente — cuéntame

---

#### Block B: Metacognición y Hábitos de Estudio (5 preguntas) [→ metacognition, study_habits]

**B1 (escenario):** "Estás estudiando un tema y de repente te das cuenta de que llevas 20 minutos 'leyendo' sin entender nada. ¿Qué haces normalmente?"
1. Sigo leyendo, a veces algo se queda
2. Retrocedo al último punto que entendí y empiezo de nuevo
3. Cambio de tema o tomo un descanso
4. Busco una explicación diferente (video, ejemplo, etc.)
5. Algo diferente — cuéntame
- Map: `metacognition.self_monitoring` (option 2 or 4 → high: 0.7–0.9; 1 or 3 → low: 0.2–0.4; 5 → interpret from free text)

**B2 (escenario):** "Para preparar un examen, ¿cuál de estas estrategias usas más?"
1. Releer mis apuntes varias veces
2. Hacerme preguntas o resolverme tests
3. Resumir con mis propias palabras
4. Repasar lo que ya sé y saltar lo que no
5. Algo diferente — cuéntame
- Map: `study_habits.retrieval_practice_awareness` (option 2 or 3 → high; 1 or 4 → low; 5 → interpret from free text)

**B3 (escala):** "¿Qué tan bien cumples tus propios planes de estudio? (ej. 'hoy voy a estudiar 1 hora de algoritmos')"
1. 1 — Casi nunca los cumplo
2. 2
3. 3 — A medias
4. 4
5. 5 — Casi siempre los cumplo
- Map: `study_habits.self_regulation`

**B4 (escenario):** "Si hoy estudias un tema, ¿cuándo crees que es mejor repasarlo para que no se te olvide?"
1. Mañana
2. En 2–3 días
3. La próxima semana
4. Cuando aparezca en el roadmap de nuevo
5. No lo había pensado — algo diferente — cuéntame
- Map: `study_habits.spacing_awareness` (option 2 or 3 → high: 0.7–0.9; 1 or 4 → low: 0.2–0.4; 5 → interpret from free text)

**B5 (escala):** "Cuando algo no te está funcionando en el estudio, ¿qué tan seguido cambias de estrategia?"
1. 1 — Casi nunca, sigo con lo mismo
2. 2
3. 3 — A veces
4. 4
5. 5 — Frecuentemente adapto mi enfoque
- Map: `metacognition.strategy_awareness`

---

#### Block C: Motivación y Mentalidad (5 preguntas) [→ motivation, mindset]

**C1 (ranking):** "De las siguientes necesidades, ¿cuál es más importante para ti en tu proceso de aprendizaje? Elige la que más resuene contigo."
1. Sentir que tengo control sobre cómo y cuándo estudio (autonomía)
2. Ver que estoy mejorando y dominando los temas (competencia)
3. Aprender junto a alguien o sentir apoyo (conexión)
4. Me identifico con las tres por igual
5. Algo diferente — cuéntame
- Map: 1 → `autonomy_need: 0.8, competence_need: 0.4, relatedness_need: 0.3`; 2 → `competence_need: 0.8, autonomy_need: 0.4, relatedness_need: 0.3`; 3 → `relatedness_need: 0.8, autonomy_need: 0.4, competence_need: 0.4`; 4 → all 0.5; 5 → interpret from free text

**C2 (escenario):** "Sacas 72% en un quiz. ¿Cuál es tu primera reacción?"
1. '¡Bien! ¿Qué necesito aprender para llegar al 85%?'
2. '¿Cómo me fue comparado con lo que se espera?'
3. 'Pasé, suficiente por ahora'
4. 'Casi me va mal, mejor no intentarlo de nuevo pronto'
5. Algo diferente — cuéntame
- Map: 1 → `motivation.orientation: "mastery"`; 2 → `"performance_approach"`; 3 → `"performance_avoidance"`; 4 → `"performance_avoidance"`; 5 → interpret from free text

**C3:** "¿Por qué quieres aprobar el EGEL? ¿Con cuál de estas opciones te identificas más?"
1. Lo necesito para titularme — es un requisito que tengo que cumplir
2. Quiero mejorar mis oportunidades laborales o profesionales
3. Quiero demostrarme a mí mismo/a que puedo hacerlo
4. Me lo están pidiendo (familia, universidad, trabajo) y hay presión externa
5. Algo diferente — cuéntame
- Map to `motivation.regulation_type`: 1 → `"introjected"`; 2 → `"identified"`; 3 → `"intrinsic"`; 4 → `"external"`

**C4 (escenario):** "Un amigo te dice: 'La inteligencia en programación es innata — o eres bueno o no.' ¿Qué piensas?"
1. Tiene razón, hay personas con talento natural
2. Tiene algo de razón pero el esfuerzo también importa
3. No estoy de acuerdo — cualquiera puede mejorar con práctica
4. No lo sé, a veces me pregunto lo mismo sobre mí
5. Algo diferente — cuéntame
- Map: `mindset.growth_belief` (3 → 0.9; 2 → 0.6; 1 or 4 → 0.3; 5 → interpret from free text)

**C5 (escenario):** "Llevas 3 sesiones seguidas fallando el mismo tema. ¿Qué haces?"
1. Me frustro y lo evito por un tiempo
2. Sigo intentándolo exactamente igual
3. Busco una explicación completamente diferente
4. Me digo que no sirvo para eso y paso a otro tema
5. Algo diferente — cuéntame
- Map: `mindset.failure_response` (3 → `"persistence"`; 1 → `"avoidance"`; 4 → `"helplessness"`; 2 → `"persistence"`; 5 → interpret from free text)
- Map: `mindset.attribution_style` (3 → `"effort"`; 4 → `"ability"`; 1 or 2 → `"effort"`)

---

#### Block D: Ansiedad y Retroalimentación (4 preguntas) [→ anxiety_profile, feedback_profile]

**D1 (escala):** "En una escala del 1 al 5, ¿cuánta ansiedad sientes cuando piensas en presentar el EGEL?"
1. 1 — Ninguna, me siento tranquilo/a
2. 2
3. 3 — Algo de nervios, manejable
4. 4
5. 5 — Mucha ansiedad, me paraliza pensar en ello
- Map: `anxiety_profile.trait_exam_anxiety` (score/5 as 0.0–1.0)

**D2 (escenario):** "Sacas 45% en un quiz. ¿Cómo te recuperas?"
1. En minutos — lo proceso y sigo adelante
2. En unas horas — necesito un descanso pero vuelvo
3. Al día siguiente — me afecta más de lo que quisiera
4. Me cuesta varios días sacudírmelo
5. Algo diferente — cuéntame
- Map: `anxiety_profile.failure_recovery_speed` (1 → `"fast"`; 2 → `"moderate"`; 3 or 4 → `"slow"`; 5 → interpret from free text)

**D3 (escenario):** "Acabas de responder una pregunta incorrectamente. ¿Qué tipo de retroalimentación prefieres?"
1. Directo al punto: 'Incorrecto. La respuesta es X porque Y.'
2. Primero reconoce lo que hice bien, luego corrige
3. Dame una pista para que yo llegue a la respuesta correcta
4. Explícame detalladamente el razonamiento correcto
5. Algo diferente — cuéntame
- Map: 1 → `directness: 0.8, emotional_support: 0.2, hint_preference: 0.2`; 2 → `directness: 0.4, emotional_support: 0.8`; 3 → `hint_preference: 0.8, directness: 0.4`; 4 → `detail_level: 0.8`; 5 → interpret from free text

**D4:** "¿Hay algo específico del EGEL o de los temas que te genere más ansiedad o te bloquee? Selecciona todo lo que aplique (puedes escribir varios números):"
1. Un tema concreto (algoritmos, redes, bases de datos, etc.)
2. El tiempo límite del examen
3. El formato de opción múltiple — siento que sé el tema pero fallo en el examen
4. No saber si estoy suficientemente preparado/a
5. La presión de presentarlo frente a otros o las consecuencias de no pasar
6. Nada en especial — me siento tranquilo/a al respecto
7. Algo diferente — cuéntame
- Map: `anxiety_profile.anxiety_triggers` (collect selected option labels as array; option 7 → append free text)

**Dynamic deepening [O7, O10]:** If D1 ≥ 4, add up to 2 follow-ups (always with numbered options + escape):

Follow-up 1: "¿En qué momento sientes más ansiedad relacionada con el EGEL?"
1. Antes — solo de pensar en que tengo que presentarlo
2. Durante el estudio — cuando siento que no avanzo
3. Al momento de contestar preguntas, aunque haya estudiado
4. Cuando veo los resultados — el miedo es a fallar en ese momento
5. Algo diferente — cuéntame

Follow-up 2: "¿Has tenido alguna estrategia que te haya ayudado con la ansiedad en exámenes antes?"
1. Sí — técnicas de respiración o relajación
2. Sí — estudiar mucho para sentirme seguro/a
3. Sí — hablarme a mí mismo/a con autocompasión
4. No he encontrado algo que realmente funcione
5. Algo diferente — cuéntame

---

#### Block E: Perfil Cognitivo y Logística (4 preguntas) [→ cognitive_profile, scaffolding]

**E1:** "¿Cuánto tiempo puedes concentrarte activamente en un tema antes de que tu mente empiece a divagar?"
1. 10–15 minutos
2. 20–30 minutos
3. 30–45 minutos
4. 45+ minutos
5. Depende mucho del día — algo diferente — cuéntame
- Map: `cognitive_profile.focus_duration_minutes` (1→15, 2→25, 3→40, 4→50; 5 → use 25 as default and note variability)

**E2:** "¿Cuánto tiempo al día puedes dedicar a estudiar para el EGEL?"
1. 30 minutos
2. 1 hora
3. 1.5–2 horas
4. 3+ horas
5. Varía mucho según el día — algo diferente — cuéntame
- Map: `cognitive_profile.daily_available_minutes` (1→30, 2→60, 3→105, 4→180; 5 → use 60 as default and note variability)

**E3 (escenario):** "Estás aprendiendo un algoritmo de ordenamiento que nunca has visto. ¿Qué preferirías que hiciera el tutor?"
1. Mostrarme un ejemplo completo resuelto paso a paso, y luego yo intento uno similar
2. Hacerme preguntas que me guíen a descubrirlo por mi cuenta
3. Darme el concepto y dejarme explorar con mis propias preguntas
4. Explicarme la teoría completa primero, con definiciones formales
5. Algo diferente — cuéntame
- Map: zpd_level: 1 → `"modeled"`, 2 → `"guided"`, 3 → `"independent"`, 4 → `"structured"`; 5 → interpret from free text
- Map: `cognitive_profile.load_tolerance`: 1 or 4 → 0.3 (prefers structure), 2 → 0.5, 3 → 0.7 (comfortable with ambiguity)

**E4 (escenario):** "Después de una sesión difícil en la que te costó trabajo pero al final entendiste, ¿qué prefieres que haga el tutor?"
1. Celebrarlo con entusiasmo — ¡me gusta el reconocimiento!
2. Una confirmación breve está bien, no necesito mucho
3. Dame un resumen de lo que aprendí, eso me motiva más
4. Pasemos rápido al siguiente tema, el progreso es lo que importa
5. Algo diferente — cuéntame
- Map: `celebration_preference` (1→0.9, 2→0.3, 3→0.5, 4→0.2; 5 → interpret from free text)
- Map: `explanation_depth` (3 → `"detailed"`, 4 → `"minimal"`, else → `"moderate"`)

---

### 3b. Dynamic Deepening on Weaknesses [O7]

After each block, if responses reveal:
- Anxiety, confusion, or significant gap → add 1-2 targeted follow-ups
- Low confidence in specific areas (Block A) → add 2-3 follow-ups per weak area
- High exam anxiety (D1 ≥ 4) → add 2 follow-ups from Block D

Total interview range: 25-35 questions depending on deepening triggers.

### 4. Silent Analysis [O4, O8]

After all questions, internally map responses to psyche.json v2 fields:

**Self-efficacy:** Convert 1-5 scale responses to 0.0–1.0 by dividing by 5.

**Strategy selection based on profile:**
- `default_pedagogy`:
  - `zpd_level: "modeled"` → `"example_first"`
  - `zpd_level: "guided"` → `"socratic_guided"`
  - `zpd_level: "independent"` → `"minimal_scaffolding"`
  - `zpd_level: "structured"` → `"step_by_step"`
- `frustration` strategy:
  - `failure_response: "helplessness"` → `"micro_step_and_celebrate"`
  - `failure_response: "avoidance"` → `"switch_topic"`
  - `failure_response: "persistence"` → `"switch_to_analogy"`
- `high_anxiety` strategy:
  - `trait_exam_anxiety > 0.7` → `"breathing_and_reduce_scope"`
  - else → `"reduce_scope_and_celebrate"`
- `overconfidence` strategy:
  - `calibration_gap > 0.15` (post-quiz) → `"challenge_with_edge_cases"`
- `plateau` strategy → always `"introduce_interleaving"`
- `streak_broken` strategy → always `"normalize_and_reconnect"`

**Initialize confidence_calibration:** Set `by_area[X].predicted` = `self_efficacy.by_area[X]` (initial self-assessment serves as baseline prediction).

**preferred_chunk_size:** derived from `load_tolerance`:
- load_tolerance < 0.35 → `"small"` (1-2 concepts per exchange)
- load_tolerance 0.35–0.65 → `"medium"` (3-4 concepts)
- load_tolerance > 0.65 → `"large"` (4-5 concepts)

### 5. Write psyche.json [O4, O8, G4]

**IMMEDIATELY after analysis — do NOT ask the user for confirmation.** Overwrite `user_profile/psyche.json` with the complete v2 profile populated from responses. Set `status: "active"`, `schema_version: 2`, fill all fields. Use today's date for `onboard_date` and `last_updated`.

### 6. Initialize progress.json [O5, O6, G4]

Read `knowledge_base/index_map.json`. For each of the 15 topics, create an entry:

```json
{
  "user_name": "<name>",
  "assessment_date": "<today>",
  "current_date": "<today>",
  "study_streak": 0,
  "last_study_date": "",
  "stress_level": "bajo",
  "total_sessions": 0,
  "topics": {
    "<topic-id>": {
      "score": 0,
      "attempts": 0,
      "last_quiz_date": "",
      "status": "sin_iniciar",
      "times_failed_consecutively": 0,
      "egel_area": "<1-4>",
      "predicted_score": null
    }
  },
  "roadmap_order": ["<ordered topic IDs>"],
  "notes": []
}
```

**Roadmap generation logic:**
- All topics start with `attempts == 0` → priority 5
- Use self-efficacy by area to order within priority 5: lowest `self_efficacy.by_area` values → earliest in roadmap
- Respect prerequisites: never place a topic before its prerequisites
- Fallback beginner topics (discrete-mathematics, programming-languages) always available

### 7. Log Session [G3]

Append to `progress/sessions.jsonl`:
```json
{"date":"<YYYY-MM-DD>","skill":"/onboard","topic":"all","details":"profile-generated","metrics":{"trait_anxiety":"<0.0-1.0>","zpd_level":"<level>","default_pedagogy":"<strategy>"}}
```

### 8. Closing Message [G1]

Summarize in Spanish with 3-4 bullets:
- Their scaffolding level and focus duration
- Their top motivational driver
- Their initial roadmap focus area (based on lowest self-efficacy)
- A motivational note tailored to their `motivation.regulation_type` and `mindset.growth_belief`

End with: **"Tu tutor está listo. Ejecuta `/study` para comenzar tu primera sesión."**
