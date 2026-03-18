---
name: jira-analyst-skill
description: Analyzes Jira tickets and produces functional or technical proposals. Use when the user asks to analyze a ticket (e.g. MDCS-456, JIRA-123), understand a problem, design solutions, or analyze CMM logs/interfaces. Escalates through Jira, Confluence, and code before asking. Generates a markdown analysis file in ~/Desktop/Análisis/. Always respond in Spanish.
license: MIT
compatibility: Requires mcp-atlassian MCP server configured with Jira and Confluence credentials. Run setup/mcp-setup.sh to configure.
metadata:
  author: facuperezbri
  version: "1.0.0"
---

# Jira Solution Designer

Self-contained skill for analyzing Jira tickets. Produces functional analysis (business rules, flows, dependencies) or technical analysis (files, code, implementation) depending on context. All required knowledge is in this file.

## Response language

**Always respond in Spanish**, regardless of the language used in the Jira ticket, Confluence, or code. All analysis output, questions, and proposals must be written in Spanish.

---

## Triggers

- "Analiza el ticket MDCS-456"
- "Dame posibles soluciones para JIRA-123"
- "¿Qué está pasando con MDCS-789?"
- "Análisis funcional de JIRA-456"
- "Analiza MDCS-123 con contexto de HU anteriores"
- "Analiza el ticket con dependencias CMM"
- *(user pastes XML/SOAP/JSON/logs alongside a ticket key)*

---

## Analysis mode detection

Determine the mode before starting:

### FUNCTIONAL mode (default)

Use when:
- The ticket is about business rules, flows, requirements, CMM behavior
- The user does not explicitly ask for code changes
- The ticket involves backend/CMM services without associated frontend code
- The user asks to "understand what's happening" or "analyze the problem"

**Output**: business analysis, rules, flows, dependencies. Does NOT include candidate files or implementation plan.

### TECHNICAL mode

Use when:
- The user explicitly asks for code changes, implementation, or technical solution
- The ticket is clearly frontend implementation (UI, components, Redux)
- The user says "what files need to be touched", "how do I implement it", "give me the technical solution"

**Output**: functional analysis + candidate files, approaches, implementation plan.

---

## Environment detection

Detect which tools are available:

1. **Local code**: Check if `src/` exists in the working directory (use Glob)
   - If it exists → use local filesystem (Glob/Grep/Read). Do NOT use Bitbucket MCP.
   - If it does NOT exist → use Bitbucket MCP if available (`bitbucket_browse`, `bitbucket_read`)

---

## Provided artifacts detection

Before starting the flow, check if the user included artifacts in their message:

### Inline artifacts (pasted in chat)

Detect if the user message contains:
- **XML/SOAP**: tags `<soapenv:`, `<cmm:`, `<ns2:`, `<request>`, `<response>`, or any structured XML
- **JSON**: objects `{...}` or arrays `[...]` with payload structure
- **Logs**: lines with timestamps, stack traces, error messages, HTTP status codes
- **CMM interfaces**: operation definitions, fields, data types

If artifacts are detected:
1. Parse and identify the artifact type (request, response, error, interface)
2. Extract key information: invoked operation, fields sent/received, error codes, CMM messages
3. Incorporate into analysis as **concrete evidence** (no need to search for it elsewhere)
4. If it's request+response, analyze: Is the request well-formed? Does the response indicate error? What fields are missing or unexpected?

### Trace number

If the user mentions a trace number (e.g. "trace 123456", "trace number: ABC-789"):
1. Ask the user to find the trace in their trace viewer and paste the CMM request and response blocks (SOAP/XML) directly in the chat.
2. Explain what to look for: the CMM request and CMM response blocks relevant to the problem.

---

## Workflow (strict order)

### Step 1: Get ticket from Jira

Call `jira_get_issue` with the issue key. Fields: `summary,description,status,priority,assignee,labels,issuetype,created,updated,parent`. Use `expand: "renderedFields"` if description is HTML. Optionally `comment_limit: 5` for recent comments.

### Step 2: Evaluate context completeness

Classify the ticket as:

- **Clear** — has target feature/flow, current/expected behavior, and at least one acceptance criterion
- **Partial** — has some elements but not all
- **Insufficient** — one line or vague (e.g. "Fix loan simulator")

**Minimum threshold** (all must be present to skip to Step 4):
- Target feature, flow, or service
- Current behavior or problem
- Desired behavior or business outcome
- At least one acceptance criterion or success signal

**Note**: Artifacts provided by the user (logs, payloads, interfaces) count as evidence for threshold evaluation. A vague ticket + detailed logs can be sufficient.

If the ticket is **clear** → skip to **Step 4**.
If it is **partial** or **insufficient** → continue to **Step 3**.

### Step 3: Progressive context escalation

Execute levels in order. After each level, re-evaluate the minimum threshold. If reached → skip to **Step 4**. If not → continue to the next level.

#### Level 1: Related tickets in Jira

Search for functional context in previous tickets from the same flow.

**JQL searches (in priority order):**

1. **Same Epic/parent**: `parent = <PARENT_KEY>` or `"Epic Link" = <EPIC_KEY>`
   - Fields: `summary,description,status,issuetype,resolution,updated,labels`
   - Limit: 20 results

2. **Same labels**: `labels IN (<LABELS>) AND project = <PROJECT> AND key != <KEY> ORDER BY updated DESC`
   - Limit: 15 results

3. **Same feature/service by text**: `project = <PROJECT> AND summary ~ "<TERM>" AND key != <KEY> AND status in (Done, Closed, Resolved) ORDER BY updated DESC`
   - Limit: 10 results

**For each relevant ticket, extract:**

| Field | What to look for |
|-------|------------------|
| Business rule | Functional requirement inferred from the ticket |
| Change made | Documented resolution or implementation |
| Risk/restriction | Edge cases, limitations, "do not do X" |

→ Re-evaluate threshold. If reached → **Step 4**.

#### Level 2: Confluence

Search for functional documentation of the feature, service, or Epic.

- `confluence_search` with terms: feature/service name, Epic name, key ticket terms
- `confluence_get_page` for the most relevant pages (max 3)
- Extract: business rules, documented flows, functional specs, CMM interfaces, diagrams

→ Re-evaluate threshold. If reached → **Step 4**.

#### Level 3: Code inspection (only if TECHNICAL mode or if it adds functional context)

**Only execute if:**
- The mode is TECHNICAL, OR
- The code can add functional context (e.g. understand validations, flows, field mapping)

**If local code exists:**
- `Glob` to locate feature files: `src/features/<feature-name>/**/*`
- `Grep` to search for key ticket terms in the code
- `Read` to inspect relevant files

**If Bitbucket is available:**
- `bitbucket_browse` to navigate the structure
- `bitbucket_read` to read relevant files

**Extract**: current flow, invoked API services, payloads, validations, field mapping.

→ Re-evaluate threshold. If reached → **Step 4**.

#### Level 4: Questions to the user

If the threshold is not reached after the previous levels, formulate 3-5 high-value questions.

**Prioritized categories:**
1. **Business objective** — What outcome is expected? Who benefits?
2. **Current vs expected behavior** — What happens today? What should happen?
3. **Affected flow** — Which screen, service, or process? Which route?
4. **Acceptance criteria** — How do we know the change is correct?
5. **Dependencies** — Backend, API, CMM service, configuration?

**Branches by ticket type:**

| Type | Priority questions |
|------|-------------------|
| Bug | Reproduction steps, frequency, error messages, environment, expected correct behavior |
| UI | Exact screen/component, mockup/visual reference, copy changes, before/after |
| Service/API | Source of truth, request/response payload, error handling, new or existing endpoint |
| CMM/Backend | CMM operation, expected interface, error codes, do you have the trace number? |
| Vague ticket | Business intent, functional area, definition of "done", who has more details |

**If the problem involves CMM and there are no artifacts**, ask specifically:
- "¿Tenés el número de trace de la transacción que falla?"
- "¿Podés pegar el request y response CMM (SOAP/XML) directamente acá?"
- "¿Podés buscar el trace en tu trace viewer y pegar los request/response CMM directamente acá?"

Wait for user responses. After receiving them, re-evaluate.

→ If reached → **Step 4**.
→ If not reached → **Level 5**.

#### Level 5: Declare blockage (Mode C)

If the minimum threshold is not reached after all levels → produce output in **Mode C** and generate markdown file.

### Step 4: Complementary enrichment

Once the minimum threshold is met, enrich with additional information:

- `jira_get_issue_development_info` — associated PRs, branches, commits
- `jira_get_issue_images` — attached mockups or screenshots
- `jira_get_issue_dates` — status history and transitions

**If the user requested extended analysis** ("con contexto de HU anteriores", "análisis funcional", "dependencias CMM"):
- Execute Level 1 searches if not already done
- Synthesize functional antecedents in a table
- Identify external dependencies (CMM, backend, contracts)

**If TECHNICAL mode and code was not inspected in Level 3:**
- Locate and read files for the affected feature
- Identify: components, reducers, sagas, API services, validations
- Search for similar patterns in existing features

### Step 5: Final confidence validation

Evaluate whether to proceed with confidence. **Blocking conditions** (if any apply → Mode C):

- No way to reproduce the problem (missing steps, environment, or data)
- Multiple valid flows and the ticket does not identify which one
- Missing expected backend response or payload contract
- Missing logs or error messages needed to locate the failure
- Missing functional definition or business rule to choose between valid options
- User responses remain ambiguous or contradictory

If blocked → **Mode C**.
If sufficient confidence → **Step 6**.

### Step 6: Produce proposal and generate markdown file

Select the appropriate output mode according to analysis mode (functional/technical) and generate both the response and the markdown file.

---

## Output modes

### Mode A-Functional: Functional analysis (sufficient context, no code)

```
## 1. Resumen de negocio
- Qué se pide o qué problema se reporta (una oración)
- Usuario/rol afectado
- Impacto esperado

## 2. Flujo afectado
- Descripción del flujo de negocio involucrado
- Servicios o sistemas que participan (CMM, backend, frontend)
- Puntos de integración relevantes

## 3. Análisis del problema
- Qué se entiende del problema a partir de la evidencia disponible
- Si hay artefactos (logs, payloads): análisis detallado de qué muestran
- Comportamiento actual vs esperado

## 4. Reglas de negocio identificadas
- Reglas explícitas (del ticket o documentación)
- Reglas inferidas (marcadas como [INFERIDA])
- Validaciones o condiciones relevantes

## 5. Diagnóstico o propuesta funcional
- Causa probable del problema / qué se necesita cambiar funcionalmente
- Opciones si hay más de un camino
- Recomendación

## 6. Dependencias y riesgos
- Dependencias externas (CMM, backend, otros equipos)
- Riesgos identificados
- Suposiciones (etiquetadas como [SUPOSICION])

## 7. Próximos pasos recomendados
- Acciones concretas para avanzar
- A quién consultar si aplica
- Qué evidencia adicional obtener si aplica
```

### Mode A-Technical: Technical proposal (sufficient context, with code)

```
## 1. Resumen de negocio
- Qué se pide (una oración)
- Usuario/rol afectado
- Impacto esperado

## 2. Áreas impactadas
- Features involucrados (nombre + path)
- Reducers, sagas, servicios API
- Componentes principales

## 3. Archivos candidatos
- Lista con rutas: src/features/..., src/api/..., etc.
- Breve motivo por archivo

## 4. Enfoques posibles (1-3)
Para cada enfoque:
- Descripción breve
- Pros/cons
- Complejidad estimada (baja/media/alta)

## 5. Enfoque recomendado
- Por qué este enfoque
- Pasos concretos (ordenados)

## 6. Riesgos e incógnitas
- Requisitos faltantes o ambiguos
- Dependencias externas
- Suposiciones asumidas (etiquetadas como [SUPOSICION])

## 7. Plan de pruebas
- Casos a validar
- Datos de prueba necesarios

## 8. Nombre de rama y commit
- Rama: feature/JIRA-XXX o fix/JIRA-XXX
- Ejemplo commit: feat(scope): descripción en español
```

### Mode B: Missing context, questions first

```
## Resumen del ticket
- Lo que el ticket establece (breve)
- Qué está claro y qué no

## Contexto faltante
- Elementos del umbral mínimo que faltan
- Por qué bloquean una propuesta concreta

## Preguntas para destrabar el análisis
- 3-5 preguntas numeradas, concretas
- Priorizadas por impacto
- Si involucra CMM: pedir trace number o request/response

## Suposiciones tentativas (opcional)
- Marcadas como [NO CONFIRMADO]
- No tratarlas como requisitos

## Próximo paso recomendado
- Qué debe hacer el usuario
- A quién consultar (PO, diseñador, QA, backend) si aplica
```

### Mode C: Blocked after investigation

```
## Resumen de lo investigado
- Qué se leyó en Jira
- Qué se encontró en Confluence (si aplica)
- Qué código se inspeccionó (si aplica)
- Qué artefactos proporcionó el usuario (si aplica)
- Qué preguntas se hicieron y qué respuestas se obtuvieron

## Por qué no puedo avanzar con confianza
- Declaración explícita: "No tengo suficiente contexto para proponer una solución confiable."
- Cuál condición de bloqueo aplica

## Evidencia faltante
Clasificar cada ítem como funcional, técnico u observable:
- Funcional: definición, regla de negocio, criterio aceptación, alcance
- Técnico: payload request/response, contrato servicio, interfaz CMM, spec API
- Observable: logs, mensajes error, pasos reproducción, screenshots, traces

Para dependencias externas usar formato:
[Dependencia externa] <tipo>: <descripción>
- Tipo: contrato | request | response | regla_negocio | observable | build
- Artefacto faltante: <qué se necesita>
- Quién puede proveerlo: <rol o sistema>

## Preguntas o artefactos necesarios
- Preguntas concretas o artefactos que destrabarían el análisis

## Quién podría proveerlo
- PO, QA, equipo backend, logs/APM, Confluence, etc.

## Próximo paso recomendado
- Qué obtener y proveer antes de solicitar nuevo análisis
```

### Mode D: Extended functional analysis + history

Use when the user asks for "con contexto de HU anteriores", "análisis funcional extendido", or "dependencias CMM".

```
## 1. Antecedentes funcionales relevantes

| Key | Summary | Regla de negocio | Cambio realizado | Riesgo/restricción |
|-----|---------|------------------|------------------|--------------------|
| MDCS-xxx | ... | ... | ... | ... |

## 2. Reglas de negocio inferidas del historial
- Síntesis de reglas de los antecedentes
- Marcar como (inferida) cuando no sea explícita
- Ej: "El monto mínimo debe validarse contra el producto seleccionado (inferida de MDCS-320)"

## 3. Dependencias externas pendientes
- Para cada dependencia: tipo, artefacto faltante, quién puede proveerlo
- Usar formato [Dependencia externa] del Modo C

## 4. Evidencia faltante para cerrar análisis
- Clasificar: funcional, técnico, observable
- Si no hay: "Evidencia suficiente para avanzar"

## 5. Síntesis y próximo paso
- Si hay contexto suficiente → producir Modo A-Funcional o A-Técnico según corresponda
- Si hay bloqueos → cerrar con Modo C
```

---

## CMM artifact analysis

When the user pastes CMM artifacts (SOAP/XML, JSON, logs), apply this specific analysis:

### CMM Request
- Identify the invoked operation (service name, method)
- List fields sent and their values
- Detect empty, null, or suspicious-format fields
- Compare against expected interface if available

### CMM Response
- Identify if success or error
- If error: extract CMM code (`cmmCode`), message (`cmmMsg`), detail (`detail`), source
- Analyze what the error means in the flow context
- If success: verify that response fields are coherent with expectations

### Request + Response together
- Trace the flow: what was sent → what was received → is it coherent?
- Identify discrepancies between sent and response
- Propose probable cause of error or unexpected behavior

### CMM Interface/Contract
- Map required vs optional fields
- Identify data types and constraints
- Compare against actual request/response if available

---

## Output file

**A markdown file is always generated** with the full analysis, regardless of mode.

### CRITICAL: Output location

**NEVER generate analysis files inside the project repository.** Analysis files would accumulate unnecessarily and clutter the application codebase. The output must **always** be written outside the repo.

**Output path (mandatory):**
- Always → `~/Desktop/Análisis/<JIRA-KEY>-analisis.md`

**File structure:**

```markdown
# Análisis: <JIRA-KEY> — <Summary del ticket>

**Fecha**: <fecha actual>
**Modo**: A-Funcional | A-Técnico | B | C | D
**Ticket**: <JIRA-KEY>
**Estado**: <status del ticket>
**Prioridad**: <priority>
**Tipo de análisis**: Funcional | Técnico

---

<Contenido completo del modo de salida seleccionado>

---

## Fuentes consultadas
- Jira: <tickets consultados>
- Confluence: <páginas consultadas> (si aplica)
- Código: <archivos inspeccionados> (si aplica)
- Bitbucket: <archivos consultados> (si aplica)
- Artefactos del usuario: <qué proporcionó> (si aplica)
```

Create the output directory if it does not exist before writing the file.

---

## Rules

1. **Always respond in Spanish** — All analysis output, questions, proposals, and file content must be written in Spanish, regardless of Jira/Confluence/code language.
2. **Functional first** — The default is functional analysis. Only go to code if the user asks or the ticket is clearly about implementation.
3. **Never invent** — If the ticket is ambiguous or requirements are missing, escalate through levels. Do not invent details.
4. **Escalation before questions** — Investigate Jira, Confluence (and code if applicable) BEFORE asking the user.
5. **User artifacts are first-class evidence** — Logs, payloads, interfaces pasted in chat have the same validity as Jira data or code.
6. **Request CMM artifacts proactively** — If the problem involves CMM services and there is no technical evidence, ask for trace number or for them to paste request/response.
7. **Label assumptions** — Every assumption must be explicitly marked as `[SUPOSICION]` or `[NO CONFIRMADO]`.
8. **Always generate file** — Generate the markdown output file in all cases, without exception.
9. **Output outside repo only** — NEVER write analysis files inside the project repository. Always use `~/Desktop/Análisis/<JIRA-KEY>-analisis.md`. Analysis files do not belong in the app codebase.
10. **Concise proposal** — 1-2 pages unless tickets are very complex.
11. **Project terminology** — Use feature names, services, and flows as they appear in Jira, Confluence, and code.
12. **Code conventions** (technical mode only) — Branch naming `feature/JIRA-ID` or `fix/JIRA-ID`, commits in Spanish.

## Optional references (NOT read by default)

These files may be read manually if additional context is needed:

- `docs/ai/feature-map.md` — feature map and routes
- `docs/ai/context-index.json` — generated context index
- `docs/ai/examples/*.md` — 4 analysis examples
- `docs/ai/clarification-playbook.md` — question playbook (deprecated, integrated here)
- `docs/ai/analysis-template.md` — analysis template (deprecated, integrated here)
- `docs/ai/history-heuristics.md` — search heuristics (deprecated, integrated here)
- `docs/ai/external-evidence-model.md` — external evidence model (deprecated, integrated here)
