# Adaptive-EGEL

Sistema de tutoría inteligente para el examen **EGEL Plus COMPU**. Transforma guías de estudio estáticas en una experiencia de aprendizaje personalizada y adaptativa basada en ciencia cognitiva con respaldo empírico.

Compatible con **Claude Code**, **Open Code**, **Gemini CLI** y **OpenAI Codex** _(sin pruebas en Codex aún)_.

---

## ¿Qué es?

Adaptive-EGEL ayuda a estudiantes de Ciencias Computacionales en México a prepararse para el EGEL Plus COMPU (Examen General para el Egreso de la Licenciatura en Ciencias Computacionales) mediante:

- **Sesiones de estudio personalizadas** que se adaptan a tu nivel, motivación y perfil de ansiedad
- **Hoja de ruta dinámica** que prioriza temas según tus puntajes, fallas y prerrequisitos
- **Cuestionamiento socrático** en lugar de lectura pasiva — descubres conceptos respondiendo preguntas guiadas
- **Perfilado basado en evidencia** fundamentado en la teoría de autoeficacia (Bandura), la Teoría de la Autodeterminación (Deci & Ryan), la ZPD de Vygotsky y el monitoreo metacognitivo

---

## Requisitos y configuración

Adaptive-EGEL funciona como un conjunto de _skills_ que se ejecutan dentro de un agente de IA. Elige la herramienta que prefieras:

### Opción A — Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

1. Clona el repositorio y entra al directorio:
   ```bash
   git clone https://github.com/cortezxm/adaptive-egel && cd adaptive-egel
   ```
2. Inicia una sesión:
   ```bash
   claude
   ```
3. Escribe `/onboard` — las skills se descubren automáticamente desde `.claude/skills/`.

> Modelos recomendados: `claude-sonnet-4-5` (uso diario) · `claude-opus-4-6` (sesiones complejas)

### Opción B — Open Code

```bash
curl -fsSL https://opencode.ai/install | bash
```

1. Clona el repositorio y entra al directorio:
   ```bash
   git clone https://github.com/cortezxm/adaptive-egel && cd adaptive-egel
   ```
2. Inicia una sesión:
   ```bash
   opencode
   ```
3. Escribe `/skills` en el TUI para ver las skills disponibles. Open Code las descubre automáticamente desde `.opencode/skills/` (o desde `.claude/skills/` como fallback).

> Modelos recomendados: `moonshot/kimi-k2` (sesiones de estudio y quiz) · `minimax/m2.1` (tareas ligeras como `/status`)

### Opción C — Gemini CLI

```bash
npm install -g @google/gemini-cli
```

1. Clona el repositorio y entra al directorio:
   ```bash
   git clone https://github.com/cortezxm/adaptive-egel && cd adaptive-egel
   ```
2. Inicia una sesión:
   ```bash
   gemini
   ```
3. Describe lo que quieres hacer en lenguaje natural — Gemini detecta la skill adecuada y te pide confirmación antes de activarla:
   - _"Quiero comenzar mi configuración inicial"_ → activa `onboard`
   - _"Quiero estudiar el siguiente tema de mi ruta"_ → activa `study`
   - _"Hazme un quiz sobre el tema que estudiamos"_ → activa `quiz`
   - _"Muéstrame mi progreso"_ → activa `status`

   También puedes listar las skills disponibles con `/skills list`.

> Modelos recomendados: `gemini-2.0-flash` (velocidad) · `gemini-2.0-pro` (razonamiento profundo)

### Opción D — OpenAI Codex ⚠️

> **Nota:** La integración con Codex está configurada pero **no ha sido probada**. Los archivos de skill están en `.agents/skills/` siguiendo el estándar Agent Skills. Reporta cualquier problema en los issues del repositorio.

```bash
npm install -g @openai/codex
```

1. Clona el repositorio y entra al directorio:
   ```bash
   git clone https://github.com/cortezxm/adaptive-egel && cd adaptive-egel
   ```
2. Inicia una sesión:
   ```bash
   codex
   ```
3. Escribe `/onboard` — las skills se descubren desde `.agents/skills/`.

> Modelos recomendados: `gpt-4o` (uso general) · `o3` (sesiones de quiz complejas)

---

## Inicio rápido

**1. Crea tu perfil (solo una vez):**

```
/onboard
```

Responde ~25 preguntas distribuidas en 5 bloques (contexto, metacognición, motivación, ansiedad, preferencias cognitivas) para generar tu perfil personalizado.

**2. Comienza una sesión de estudio:**

```
/study
```

El tutor selecciona el mejor tema para ti, carga el material y te guía con preguntas socráticas calibradas a tu nivel.

**3. Valida lo que aprendiste:**

```
/quiz
```

Cinco preguntas en formato EGEL sobre el tema. Tu puntaje actualiza tu hoja de ruta y hace evolucionar tu perfil.

**4. Revisa tu progreso:**

```
/status
```

Panel con rachas, temas dominados, áreas débiles y la siguiente recomendación.

---

## Referencia de skills

| Skill      | Cuándo usarla                                              |
| ---------- | ---------------------------------------------------------- |
| `/onboard` | Configuración inicial — genera tu perfil de aprendizaje    |
| `/study`   | Sesión de estudio diaria sobre tu próximo tema en la ruta  |
| `/quiz`    | Evaluación de 5 preguntas después de una sesión de estudio |
| `/status`  | Ver tu panel de progreso y racha                           |

> Las skills están definidas en `.claude/skills/<nombre>/SKILL.md` y son compatibles con Claude Code, Open Code, Gemini CLI y OpenAI Codex _(sin pruebas en Codex)_. No se requiere configuración adicional — basta con clonar el repositorio.

---

## Arquitectura

### Diseño de tres capas

```
Capa de conocimiento    → knowledge_base/ (guías de estudio + índice de temas)
Capa de estado          → user_profile/ + progress/ (psyche.json, progress.json, sessions.jsonl)
Capa de orquestación    → .claude/skills/ (skills de Claude Code sin estado)
```

**Principio clave:** Cada skill es sin estado y orientada a archivos. No hay base de datos ni backend — todo el estado vive en archivos JSON legibles por humanos que puedes inspeccionar y modificar directamente.

### Estructura de archivos

```
adaptive-egel/
├── .claude/skills/            # Fuente canónica de skills (Claude Code)
│   ├── onboard/SKILL.md      # Evaluación psicológica y de conocimiento inicial
│   ├── study/SKILL.md        # Sesión de estudio adaptativa diaria
│   ├── quiz/SKILL.md         # Evaluación dinámica + actualización de hoja de ruta
│   └── status/SKILL.md       # Panel de progreso
├── .gemini/skills/            # Symlinks → .claude/skills/ (Gemini CLI)
├── .opencode/
│   └── skills/                # Symlinks → .claude/skills/ (Open Code)
├── .agents/skills/            # Symlinks → .claude/skills/ (OpenAI Codex, sin pruebas)
├── knowledge_base/
│   ├── index_map.json         # Metadatos de temas (prerrequisitos, dificultad, área EGEL)
│   └── guias/                 # Archivos .md de guías de estudio (uno por tema)
├── user_profile/
│   └── psyche.json            # Tu perfil de aprendizaje (basado en evidencia, v2)
├── progress/
│   ├── progress.json          # Puntajes, racha, orden de la hoja de ruta
│   └── sessions.jsonl         # Historial de sesiones (solo se agrega, nunca se borra)
├── README.md
├── CLAUDE.md                  # Instrucciones para Claude Code
└── AGENTS.md                  # Instrucciones para Open Code
```

---

## El examen EGEL-COMPU

El EGEL-COMPU cubre 4 áreas:

| Área                                | Temas                                                                        |
| ----------------------------------- | ---------------------------------------------------------------------------- |
| **Área 1 — Algoritmia**             | Algoritmos, estructuras de datos, matemáticas discretas, lógica              |
| **Área 2 — Software de Base**       | Arquitectura de computadoras, sistemas operativos, compiladores, redes       |
| **Área 3 — Software de Aplicación** | Ingeniería de software, lenguajes de programación, bases de datos, seguridad |
| **Área 4 — Cómputo Inteligente**    | Inteligencia artificial, minería de datos, cómputo distribuido               |

Cada área tiene dos niveles de desempeño: **Satisfactorio** (aprobatorio) y **Sobresaliente** (distinción). El tutor cubre ambos.

---

## Tu perfil de aprendizaje (`psyche.json`)

Tu perfil se construye sobre ciencia cognitiva validada.

### Qué mide

| Dimensión                      | Basada en                                    | Cómo se usa                                                                                     |
| ------------------------------ | -------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Autoeficacia**               | Bandura (1997)                               | Baja → más andamiaje + encuadre de "esto es alcanzable"                                         |
| **Metacognición**              | Flavell / Schraw                             | Bajo automonitoreo → insertar verificaciones de comprensión                                     |
| **Motivación**                 | Teoría de la Autodeterminación (Deci & Ryan) | Necesidad de autonomía → ofrecer opciones; regulación externa → conectar con objetivos EGEL     |
| **Perfil cognitivo**           | Teoría de la Carga Cognitiva                 | Tolerancia a la carga → tamaño de fragmentos; duración de foco → sugerencias de descanso        |
| **Mentalidad**                 | Dweck (2006, refinada)                       | Patrón de indefensión → micro-pasos + celebrar; creencia de crecimiento → normalizar dificultad |
| **Perfil de ansiedad**         | Investigación sobre ansiedad ante exámenes   | Alta ansiedad rasgo → usar "práctica" en lugar de "quiz"; reducir alcance bajo estrés           |
| **Andamiaje (ZPD)**            | Vygotsky                                     | Avanza de modelado → guiado → independiente a medida que crece el dominio                       |
| **Calibración de confianza**   | Monitoreo metacognitivo                      | Sobreconfiado → preguntas más difíciles; subconfiado → revelar brecha positiva                  |
| **Efectividad de estrategias** | Analítica de aprendizaje                     | Estrategias con < 30% de efectividad tras 3 usos se marcan para reemplazo                       |

### Cómo evoluciona

Tu perfil **no es estático** — se actualiza después de cada sesión:

- `/onboard` — establece los valores iniciales desde tu entrevista de 25 preguntas
- `/study` — actualiza `state_anxiety` y `zpd_level` según el comportamiento en sesión
- `/quiz` — actualiza `self_efficacy`, `confidence_calibration`, `scaffolding.zpd_level` y `strategy_effectiveness`

**Regla de amortiguación:** Ningún campo numérico cambia más de ±0.1 por sesión para evitar oscilaciones por ruido.

---

## Lógica de la hoja de ruta

El tutor prioriza los temas automáticamente con esta fórmula:

```
puntaje < 70% Y con intentos  → prioridad 1  (necesita revisión urgente)
3+ fallas consecutivas         → prioridad 2  (activar apoyo profundo)
puntaje 70-84% Y con intentos  → prioridad 3  (progresando, necesita refinamiento)
puntaje >= 85%                 → prioridad 4  (dominado)
nunca intentado                → prioridad 5  (respetar orden original)
```

Los prerrequisitos siempre se respetan: un tema no aparecerá en tu hoja de ruta hasta que sus prerrequisitos alcancen el 60%.

Las actualizaciones de puntaje usan un **promedio ponderado** para preservar el progreso acumulado:

```
nuevo_puntaje = puntaje_anterior × 0.35 + puntaje_quiz × 0.65
```

---

## Configuración

Puedes inspeccionar y editar manualmente tus archivos de estado en cualquier momento:

- `user_profile/psyche.json` — tu perfil de aprendizaje (todos los campos numéricos de 0.0 a 1.0, enumeraciones de cadenas)
- `progress/progress.json` — puntajes por tema, orden de la hoja de ruta, racha de sesiones
- `progress/sessions.jsonl` — historial completo de cada sesión (solo se agrega)

Si deseas **reiniciar tu perfil**, elimina o restablece `user_profile/psyche.json` a `status: "bootstrap-default"` y ejecuta `/onboard` de nuevo.

---

## Decisiones de diseño

**¿Por qué estado en archivos (sin base de datos)?**

- Legible por humanos y controlable por versiones
- El usuario puede inspeccionar o modificar directamente
- Se alinea con la filosofía sin estado y orientada a archivos de Claude Code

**¿Por qué skills sin estado?**

- Cada invocación es independiente y reproducible
- Sin estado oculto entre sesiones
- Más fácil de depurar

---

## Créditos

- Estructura del examen EGEL-COMPU: CENEVAL (Centro Nacional de Evaluación para la Educación Superior)
- Inspiración original: [juanQNav/Egel-computer-science](https://github.com/juanQNav/Egel-computer-science)
- Construido con [Claude Code](https://claude.ai/code) · compatible con [Open Code](https://opencode.ai), [Gemini CLI](https://developers.google.com/gemini) y OpenAI Codex _(sin pruebas en Codex)_
