---
name: onboard
description: "Use this skill when the user runs /onboard, wants to start, set up the tutor, do the initial assessment, or begin their study journey for the first time."
---

# /onboard — Initial Psychological & Knowledge Assessment

## Purpose

One-time calibration session. Ask ~20 questions one at a time (dynamically adding more when weaknesses are detected), then IMMEDIATELY write `psyche.json` and initialize `progress.json`. No confirmation prompts — writing is mandatory and automatic.

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

## Flow

### 1. Check Idempotency [O1]

Read `user_profile/psyche.json`. If it exists and `status` is NOT `bootstrap-default`:
- Tell the user they already have a profile
- Ask if they want to recalibrate (this will overwrite their profile)
- If they say no, suggest `/study` instead and end

### 2. Welcome & Name [O3, G1]

Greet the user warmly in Spanish. Ask for their name. Explain:
- "Vamos a hacer una calibración de ~20 preguntas para personalizar tu experiencia"
- "Una pregunta a la vez, sin prisa"

### 3. Interview — One Question Per Message [O2]

Ask questions using Kolb + VARK frameworks. **One question per message, wait for response before next.**

**IMPORTANT [O9]:** Every question that offers choices MUST present them as a numbered list. Example format:
> ¿Cuánto tiempo puedes concentrarte antes de necesitar un descanso?
> 1. 15 minutos
> 2. 30 minutos
> 3. 45 minutos
> 4. 1 hora o más

Never list options inline (e.g. "a / b / c" or "a, b, c"). Always use a numbered list.

**Block: Learning Style (6 questions)**
1. When learning something new, do you prefer:
   1. Examples
   2. Theory
   3. Diagrams
   4. Hands-on practice
2. When you get stuck on a problem, what's your first reaction?
   1. Re-read the material
   2. Search for examples
   3. Ask someone
   4. Try a different approach
3. How long are your ideal study sessions?
   1. 15–30 minutes
   2. 30–60 minutes
   3. 1–2 hours
   4. 2+ hours
4. If you had to explain a sorting algorithm to a friend, how would you do it?
   1. Step-by-step
   2. Analogy
   3. Diagram
   4. Code
5. How do you feel about multiple-choice exams?
   1. Comfortable
   2. Prefer open-ended
   3. Anxious
   4. Depends on the topic
6. What's your biggest concern about the EGEL?
   1. Breadth of topics
   2. Depth needed
   3. Time pressure
   4. A specific area

**Block: VARK (4 questions)**
7. When studying, which helps you most?
   1. Reading text
   2. Watching videos
   3. Listening to explanations
   4. Doing exercises
8. What kind of feedback helps you learn?
   1. Detailed explanations
   2. Just the correct answer
   3. Encouragement first, then correction
   4. Examples of the correct approach
9. How long can you maintain focus before needing a break?
   1. 15 minutes
   2. 30 minutes
   3. 45 minutes
   4. 1 hour or more
10. Do you prefer to understand one topic deeply or cover many topics broadly first?
    1. One topic deeply
    2. Many topics broadly first

**Block: Stress & Motivation (6 questions)**
11. What motivates you most?
    1. Seeing progress stats
    2. Competing with yourself
    3. Intrinsic understanding
    4. Completing milestones
12. On a scale of 1–5, how anxious are you about the EGEL?
    1. 1 — Calm
    2. 2
    3. 3
    4. 4
    5. 5 — Very anxious
13. Have you attempted the EGEL before? If yes, how did it go? (open-ended — no numbered options)
14. When you fail a quiz, how do you typically feel?
    1. Frustrated
    2. Motivated to retry
    3. Indifferent
    4. Discouraged
15. When you're struggling, what would you want the tutor to do?
    1. Change the approach
    2. Suggest a break
    3. Show an easier example
    4. Encourage me
16. How many hours per day can you dedicate to studying?
    1. 30 minutes
    2. 1 hour
    3. 2 hours
    4. 3+ hours

**Block: Self-Assessment Baseline (4 questions)**
17. Rate your confidence in Área 1 — Algoritmia (algorithms, data structures, discrete math, logic):
    1. 1 — No confidence
    2. 2
    3. 3
    4. 4
    5. 5 — Very confident
18. Rate your confidence in Área 2 — Software de Base (architecture, OS, compilers, networks): (same 1–5 scale)
19. Rate your confidence in Área 3 — Software de Aplicación (software eng, languages, databases, security): (same 1–5 scale)
20. Rate your confidence in Área 4 — Cómputo Inteligente (AI, data mining, distributed computing): (same 1–5 scale)

### 3b. Dynamic Deepening on Weaknesses [O7]

After the self-assessment block (Q17-Q20), if ANY area has confidence ≤ 2:
- Ask 2-3 follow-up questions to understand the weakness better:
  - "¿Qué temas específicos de [área] te resultan más difíciles?"
  - "¿Has estudiado [área] formalmente o es algo que no has visto?"
  - "¿Hay algún subtema dentro de [área] que sí domines?"

Also during any block, if the user's response reveals anxiety, confusion, or a specific gap:
- Add a targeted follow-up to understand depth of the gap
- Example: if user says "algorithms stress me out" → "¿Es la parte matemática o la implementación lo que te genera más estrés?"

The total interview may be 20-28 questions depending on how many follow-ups are triggered.

### 4. Silent Analysis

After all questions, analyze responses internally:
- Map learning style → `learning_style` field (examples/theory/visual/kinesthetic)
- Map pace from session duration + focus answers
- Map motivator from Q11
- Map stress baseline from Q12 + Q14
- Map error tolerance from Q14 + Q15
- Generate personalized strategies for frustration, streak breaks, high stress
- Use self-assessment (Q17-Q20) to influence initial roadmap ordering

### 5. Write psyche.json [O4, O8, G4]

**IMMEDIATELY after analysis — do NOT ask the user for confirmation.** Read `user_profile/psyche.json`, then overwrite with complete profile:

```json
{
  "status": "active",
  "user_name": "<from Q welcome>",
  "learning_style": "<mapped>",
  "pace": "slow|medium|fast",
  "primary_motivator": "<mapped>",
  "stress_baseline": "low|medium|high",
  "error_tolerance": "low|medium|high",
  "preferred_feedback": "<mapped>",
  "focus_duration_minutes": <number>,
  "daily_study_hours": <number>,
  "egel_attempts": <0 or more>,
  "area_confidence": {
    "1_algoritmia": <1-5>,
    "2_software_base": <1-5>,
    "3_software_aplicacion": <1-5>,
    "4_computo_inteligente": <1-5>
  },
  "strategies": {
    "frustration": "<personalized>",
    "streak_broken": "<personalized>",
    "high_stress": "<personalized>"
  },
  "last_updated": "<YYYY-MM-DD>"
}
```

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
      "egel_area": <1-4>
    }
  },
  "roadmap_order": ["<ordered topic IDs>"],
  "notes": []
}
```

**Roadmap generation logic:**
- All topics start with `attempts == 0` → priority 5
- Use self-assessment confidence (Q17-Q20) to order within priority 5: lowest confidence areas first
- Respect prerequisites: never place a topic before its prerequisites
- Fallback beginner topics (discrete-mathematics, programming-languages) always available

### 7. Log Session [G3]

Append to `progress/sessions.jsonl`:
```json
{"date":"<YYYY-MM-DD>","skill":"/onboard","topic":"all","details":"profile-generated","metrics":{"stress":"<baseline>"}}
```

### 8. Closing Message [G1]

Summarize in Spanish with 3-4 bullets:
- Their learning style and pace
- Their initial roadmap focus area
- A motivational note aligned to their profile

End with: **"Tu tutor está listo. Ejecuta `/study` para comenzar tu primera sesión."**
