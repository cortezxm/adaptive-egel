---
name: onboard
description: "Use this skill when the user runs /onboard, wants to start, set up the tutor, do the initial assessment, or begin their study journey for the first time."
---

# /onboard — Initial Psychological & Knowledge Assessment

## Purpose

One-time calibration session. Ask ~20 questions one at a time, generate `psyche.json` and initialize `progress.json` with all 15 topics and a prerequisite-respecting roadmap.

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
| **O4** | Write `psyche.json` only AFTER all questions are answered (not during) |
| **O5** | Initialize `progress.json` with all 15 topics from `index_map.json` |
| **O6** | Generate `roadmap_order` respecting prerequisites |

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

**Block: Learning Style (6 questions)**
1. When learning something new, do you prefer examples, theory, diagrams, or hands-on practice?
2. When you get stuck on a problem, what's your first reaction? (re-read, search examples, ask someone, try different approach)
3. How long are your ideal study sessions? (15-30 min / 30-60 min / 1-2 hours / 2+ hours)
4. If you had to explain a sorting algorithm to a friend, how would you do it? (step-by-step, analogy, diagram, code)
5. How do you feel about multiple-choice exams? (comfortable / prefer open-ended / anxious / depends on topic)
6. What's your biggest concern about the EGEL? (breadth of topics / depth needed / time pressure / specific area)

**Block: VARK (4 questions)**
7. When studying, which helps you most? (reading text / watching videos / listening to explanations / doing exercises)
8. What kind of feedback helps you learn? (detailed explanations / just correct answer / encouragement first then correction / examples of correct approach)
9. How long can you maintain focus before needing a break? (15 min / 30 min / 45 min / 1 hour+)
10. Do you prefer to understand one topic deeply or cover many topics broadly first?

**Block: Stress & Motivation (6 questions)**
11. What motivates you most? (seeing progress stats / competing with yourself / intrinsic understanding / completing milestones)
12. On a scale of 1-5, how anxious are you about the EGEL? (1=calm, 5=very anxious)
13. Have you attempted the EGEL before? If yes, how did it go?
14. When you fail a quiz, how do you typically feel? (frustrated / motivated to retry / indifferent / discouraged)
15. When you're struggling, what would you want the tutor to do? (change approach / give a break / show easier example / encourage you)
16. How many hours per day can you dedicate to studying? (30 min / 1 hour / 2 hours / 3+ hours)

**Block: Self-Assessment Baseline (4 questions)**
17. Rate your confidence in Área 1 — Algoritmia (algorithms, data structures, discrete math, logic): (1-5)
18. Rate your confidence in Área 2 — Software de Base (architecture, OS, compilers, networks): (1-5)
19. Rate your confidence in Área 3 — Software de Aplicación (software eng, languages, databases, security): (1-5)
20. Rate your confidence in Área 4 — Cómputo Inteligente (AI, data mining, distributed computing): (1-5)

### 4. Silent Analysis

After all questions, analyze responses internally:
- Map learning style → `learning_style` field (examples/theory/visual/kinesthetic)
- Map pace from session duration + focus answers
- Map motivator from Q11
- Map stress baseline from Q12 + Q14
- Map error tolerance from Q14 + Q15
- Generate personalized strategies for frustration, streak breaks, high stress
- Use self-assessment (Q17-Q20) to influence initial roadmap ordering

### 5. Write psyche.json [O4, G4]

Read `user_profile/psyche.json`, then overwrite with complete profile:

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
