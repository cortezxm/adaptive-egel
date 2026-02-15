---
name: study
description: "Use this skill when the user runs /study, says they want to study, learn, review a topic, or asks what they should study today."
---

# /study — Adaptive Study Session

## Purpose

Daily study session. Load the next topic from the roadmap, present it adapted to the user's psychological profile, detect stress passively, and update progress state.

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
| **S3** | NEVER paste raw .md — always synthesize and adapt |
| **S4** | Label the topic with its EGEL Area at the start (e.g., "Área 1: Algoritmia") |
| **S5** | Cover both Satisfactorio AND Sobresaliente objectives |
| **S6** | Execute passive burnout detection throughout the session |
| **S7** | Update `progress.json` (streak, total_sessions, last_study_date, roadmap_order) at close |
| **S8** | Close with explicit recommendation: "Ejecuta `/quiz` para validar lo aprendido" |

## Flow

### 1. Read State [G2, G5]

Read these files in order:
1. `progress/progress.json`
2. `user_profile/psyche.json`
3. `knowledge_base/index_map.json`

If `psyche.json` has `status: "bootstrap-default"`, tell user: "Aún no has completado tu calibración. Ejecuta `/onboard` primero."

### 2. Select Topic

Take `roadmap_order[0]` from `progress.json`.

**Prerequisite validation:** Check if all prerequisites for this topic have `score >= 60` in `progress.json`. If not:
- Skip to the next topic in roadmap that has its prerequisites met
- If no topic has prerequisites met, fall back to a beginner topic with no prerequisites (discrete-mathematics or programming-languages)

### 3. Announce [S4, G1]

Start the session by announcing in Spanish:
- The EGEL area: "**Área [N]: [Area Name]**"
- The topic: "[Topic Title]"
- Brief context of why this topic is next in their roadmap

### 4. Load Content [S1]

Read the topic `.md` file from `knowledge_base/` using the path from `index_map.json`.

If the topic has prerequisites with `score < 80`, also load prerequisite `.md` files for reference (max ~50k tokens total).

If a file is very large (>30k tokens), read only the first half and note that you're focusing on core concepts.

### 5. Calibrate Pedagogy [S2]

Before presenting any content, apply `psyche.json`:
- `learning_style: "examples"` → lead with concrete examples, then theory
- `learning_style: "theory"` → lead with concepts, then examples
- `learning_style: "visual"` → use diagrams, tables, ASCII art
- `learning_style: "kinesthetic"` → use step-by-step walkthroughs, exercises
- `pace: "slow"` → shorter sections, more pauses
- `pace: "fast"` → denser content, fewer pauses
- `preferred_feedback` → calibrate encouragement style

### 6. Present Content [S3, S5, G1]

Present the material in Spanish, synthesized and adapted. **Never paste the raw .md file.** Structure:

1. **Overview** — What is this topic and why it matters for the EGEL (2-3 sentences)
2. **Core Concepts** — The key ideas, calibrated to learning style. Use the guide content as source but rewrite for the user's profile.
3. **Example or Analogy** — One concrete example or real-world analogy
4. **Key Points: Nivel Satisfactorio** — What you need to know for a passing score
5. **Key Points: Nivel Sobresaliente** — What takes you to excellence (deeper analysis, edge cases)
6. **Comprehension Check** — One ungraded question to verify understanding (not a quiz, just engagement)

### 7. Passive Burnout Detection [S6]

Throughout the session, monitor the user's messages for:
- **Frustration language**: "no entiendo", "esto es imposible", "ya me cansé", short/terse responses
- **Self-doubt**: "no soy bueno para esto", "nunca voy a pasar"
- **Disengagement**: very short responses, "ok", "sí", single words repeatedly

**If detected, adapt silently:**
- Switch from theory to analogy or real-world example
- Reduce complexity, focus on one subtopic instead of many
- Add encouraging language aligned to `psyche.json` strategies
- Do NOT announce that you're detecting burnout — just adapt naturally

### 8. Close Session [S7, S8, G3, G4]

**Takeaways:** Provide 3-4 key takeaways in Spanish.

**Update progress.json [S7, G4]:**
- `current_date` → today
- `study_streak`: if `last_study_date` is yesterday → increment; if today → keep; else → reset to 1
- `last_study_date` → today
- `total_sessions` → increment by 1
- Reorder `roadmap_order` using the priority formula:

```
score < 70 AND attempts > 0 → priority 1
times_failed_consecutively >= 3 → priority 2
score < 85 AND attempts > 0 → priority 3
score >= 85 → priority 4
attempts == 0 → priority 5
```

Respect prerequisites when ordering.

**Log session [G3]:**
Append to `progress/sessions.jsonl`:
```json
{"date":"<YYYY-MM-DD>","skill":"/study","topic":"<topic-id>","details":"session-complete","metrics":{"stress":"<current level>","streak":<number>}}
```

**Recommendation [S8]:**
End with: **"Ejecuta `/quiz` para validar lo que aprendiste sobre [Topic Title]."**

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
