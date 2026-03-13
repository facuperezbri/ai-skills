---
name: jira-analyst-skill
description: Analiza tickets de Jira y produce propuestas funcionales o técnicas. Usar cuando el usuario pide analizar un ticket (ej. MDCS-456, JIRA-123), entender un problema, diseñar soluciones, o analizar logs/interfaces CMM. Escala por Jira, Confluence y código antes de preguntar. Genera un archivo markdown con el análisis en ~/Desktop/Análisis/.
license: MIT
compatibility: Requires mcp-atlassian MCP server configured with Jira and Confluence credentials. Run setup/mcp-setup.sh to configure.
metadata:
  author: facuperezbri
  version: "1.0.0"
---

# Jira Solution Designer

Skill autocontenida para analizar tickets de Jira. Produce análisis funcionales (reglas de negocio, flujos, dependencias) o técnicos (archivos, código, implementación) según el contexto. Todo el conocimiento necesario está en este archivo.

## Disparadores

- "Analiza el ticket MDCS-456"
- "Dame posibles soluciones para JIRA-123"
- "¿Qué está pasando con MDCS-789?"
- "Análisis funcional de JIRA-456"
- "Analiza MDCS-123 con contexto de HU anteriores"
- "Analiza el ticket con dependencias CMM"
- *(usuario pega XML/SOAP/JSON/logs junto con un ticket key)*

---

## Detección de modo de análisis

Determinar el modo antes de comenzar:

### Modo FUNCIONAL (default)

Usar cuando:
- El ticket trata sobre reglas de negocio, flujos, requisitos, comportamiento CMM
- El usuario no pide explícitamente cambios de código
- El ticket involucra servicios backend/CMM sin código frontend asociado
- El usuario pide "entender qué pasa" o "analizar el problema"

**Salida**: análisis de negocio, reglas, flujos, dependencias. NO incluye archivos candidatos ni plan de implementación.

### Modo TECNICO

Usar cuando:
- El usuario pide explícitamente cambios de código, implementación, o solución técnica
- El ticket es claramente de implementación frontend (UI, componentes, Redux)
- El usuario dice "qué archivos hay que tocar", "cómo lo implemento", "dame la solución técnica"

**Salida**: análisis funcional + archivos candidatos, enfoques, plan de implementación.

---

## Detección de entorno

Detectar qué herramientas están disponibles:

1. **Código local**: Verificar si existe `src/` en el directorio de trabajo (usar Glob)
   - Si existe → usar filesystem local (Glob/Grep/Read). NO usar Bitbucket MCP.
   - Si NO existe → usar Bitbucket MCP si está disponible (`bitbucket_browse`, `bitbucket_read`)

---

## Detección de artefactos proporcionados

Antes de iniciar el flujo, revisar si el usuario incluyó artefactos en su mensaje:

### Artefactos inline (pegados en el chat)

Detectar si el mensaje del usuario contiene:
- **XML/SOAP**: tags `<soapenv:`, `<cmm:`, `<ns2:`, `<request>`, `<response>`, o cualquier XML estructurado
- **JSON**: objetos `{...}` o arrays `[...]` con estructura de payload
- **Logs**: líneas con timestamps, stack traces, mensajes de error, HTTP status codes
- **Interfaces CMM**: definiciones de operaciones, campos, tipos de datos

Si se detectan artefactos:
1. Parsear e identificar qué tipo de artefacto es (request, response, error, interfaz)
2. Extraer información clave: operación invocada, campos enviados/recibidos, códigos de error, mensajes CMM
3. Incorporar al análisis como **evidencia concreta** (no hace falta buscarla en otra fuente)
4. Si es un request+response, analizar: ¿el request está bien formado? ¿La response indica error? ¿Qué campos faltan o son inesperados?

### Número de trace

Si el usuario menciona un número de trace (ej. "trace 123456", "trace number: ABC-789"):
1. Indicar al usuario que busque el trace en su trace viewer y pegue directamente los bloques de request y response CMM (SOAP/XML) en el chat.
2. Explicar qué buscar: los bloques de CMM request y CMM response relevantes al problema.

---

## Flujo de trabajo (orden estricto)

### Paso 1: Obtener ticket de Jira

Llamar `jira_get_issue` con el key del issue. Campos: `summary,description,status,priority,assignee,labels,issuetype,created,updated,parent`. Usar `expand: "renderedFields"` si la descripción es HTML. Opcionalmente `comment_limit: 5` para comentarios recientes.

### Paso 2: Evaluar completitud del contexto

Clasificar el ticket como:

- **Claro** — tiene feature/flujo objetivo, comportamiento actual/esperado, y al menos un criterio de aceptación
- **Parcial** — tiene algunos elementos pero no todos
- **Insuficiente** — una línea o vago (ej. "Corregir simulador de préstamos")

**Umbral mínimo** (todos deben estar presentes para avanzar directo al Paso 4):
- Feature, flujo o servicio objetivo
- Comportamiento actual o problema
- Comportamiento deseado o resultado de negocio
- Al menos un criterio de aceptación o señal de éxito

**Nota**: Los artefactos proporcionados por el usuario (logs, payloads, interfaces) cuentan como evidencia para evaluar el umbral. Un ticket vago + logs detallados puede ser suficiente.

Si el ticket es **claro** → saltar al **Paso 4**.
Si es **parcial** o **insuficiente** → continuar al **Paso 3**.

### Paso 3: Escalamiento progresivo de contexto

Ejecutar los niveles en orden. Después de cada nivel, reevaluar el umbral mínimo. Si se alcanza → saltar al **Paso 4**. Si no → continuar al siguiente nivel.

#### Nivel 1: Tickets relacionados en Jira

Buscar contexto funcional en tickets previos del mismo flujo.

**Búsquedas JQL (en orden de prioridad):**

1. **Mismo Epic/parent**: `parent = <PARENT_KEY>` o `"Epic Link" = <EPIC_KEY>`
   - Campos: `summary,description,status,issuetype,resolution,updated,labels`
   - Límite: 20 resultados

2. **Mismos labels**: `labels IN (<LABELS>) AND project = <PROJECT> AND key != <KEY> ORDER BY updated DESC`
   - Límite: 15 resultados

3. **Mismo feature/servicio por texto**: `project = <PROJECT> AND summary ~ "<TERMINO>" AND key != <KEY> AND status in (Done, Closed, Resolved) ORDER BY updated DESC`
   - Límite: 10 resultados

**Para cada ticket relevante, extraer:**

| Campo | Qué buscar |
|-------|-----------|
| Regla de negocio | Requisito funcional que se desprende del ticket |
| Cambio realizado | Resolución o implementación documentada |
| Riesgo/restricción | Edge cases, limitaciones, "no hacer X" |

→ Reevaluar umbral. Si alcanza → **Paso 4**.

#### Nivel 2: Confluence

Buscar documentación funcional del feature, servicio o Epic.

- `confluence_search` con términos: nombre del feature/servicio, nombre del Epic, términos clave del ticket
- `confluence_get_page` para las páginas más relevantes (máximo 3)
- Extraer: reglas de negocio, flujos documentados, especificaciones funcionales, interfaces CMM, diagramas

→ Reevaluar umbral. Si alcanza → **Paso 4**.

#### Nivel 3: Inspección de código (solo si modo TECNICO o si aporta contexto funcional)

**Solo ejecutar si:**
- El modo es TECNICO, O
- El código puede aportar contexto funcional (ej. entender validaciones, flujos, mapeo de campos)

**Si hay código local:**
- `Glob` para localizar archivos del feature: `src/features/<nombre-feature>/**/*`
- `Grep` para buscar términos clave del ticket en el código
- `Read` para inspeccionar archivos relevantes

**Si hay Bitbucket disponible:**
- `bitbucket_browse` para navegar la estructura
- `bitbucket_read` para leer archivos relevantes

**Extraer**: flujo actual, servicios API invocados, payloads, validaciones, mapeo de campos.

→ Reevaluar umbral. Si alcanza → **Paso 4**.

#### Nivel 4: Preguntas al usuario

Si tras los niveles anteriores el umbral no se alcanza, formular 3-5 preguntas de alto valor.

**Categorías priorizadas:**
1. **Objetivo de negocio** — ¿Qué resultado se espera? ¿Quién se beneficia?
2. **Comportamiento actual vs esperado** — ¿Qué pasa hoy? ¿Qué debería pasar?
3. **Flujo afectado** — ¿Qué pantalla, servicio o proceso? ¿Qué ruta?
4. **Criterios de aceptación** — ¿Cómo sabemos que el cambio es correcto?
5. **Dependencias** — ¿Backend, API, servicio CMM, configuración?

**Ramas por tipo de ticket:**

| Tipo | Preguntas prioritarias |
|------|----------------------|
| Bug | Pasos de reproducción, frecuencia, mensajes de error, entorno, comportamiento correcto esperado |
| UI | Pantalla/componente exacto, mockup/referencia visual, cambios de copy, antes/después |
| Servicio/API | Fuente de verdad, payload request/response, manejo de errores, endpoint nuevo o existente |
| CMM/Backend | Operación CMM, interfaz esperada, códigos de error, ¿tiene el trace number? |
| Ticket vago | Intención de negocio, área funcional, definición de "terminado", quién tiene más detalles |

**Si el problema involucra CMM y no hay artefactos**, preguntar específicamente:
- "¿Tenés el número de trace de la transacción que falla?"
- "¿Podés pegar el request y response CMM (SOAP/XML) directamente acá?"
- "¿Podés buscar el trace en tu trace viewer y pegar los request/response CMM directamente acá?"

Esperar respuestas del usuario. Tras recibirlas, reevaluar.

→ Si alcanza → **Paso 4**.
→ Si no alcanza → **Nivel 5**.

#### Nivel 5: Declarar bloqueo (Modo C)

Si tras todos los niveles no se alcanza el umbral mínimo → producir salida en **Modo C** y generar archivo markdown.

### Paso 4: Enriquecimiento complementario

Una vez que el umbral mínimo se cumple, enriquecer con información adicional:

- `jira_get_issue_development_info` — PRs, branches, commits asociados
- `jira_get_issue_images` — mockups o capturas adjuntas
- `jira_get_issue_dates` — historial de estados y transiciones

**Si el usuario pidió análisis extendido** ("con contexto de HU anteriores", "análisis funcional", "dependencias CMM"):
- Ejecutar búsquedas del Nivel 1 si no se hicieron
- Sintetizar antecedentes funcionales en tabla
- Identificar dependencias externas (CMM, backend, contratos)

**Si modo TECNICO y no se inspeccionó código en Nivel 3:**
- Localizar y leer archivos del feature afectado
- Identificar: componentes, reducers, sagas, servicios API, validaciones
- Buscar patrones similares en features existentes

### Paso 5: Validación final de confianza

Evaluar si se puede avanzar con confianza. **Condiciones de bloqueo** (si alguna aplica → Modo C):

- No hay forma de reproducir el problema (faltan pasos, entorno o datos)
- Múltiples flujos válidos y el ticket no identifica cuál
- Falta respuesta esperada del backend o contrato de payload
- Faltan logs o mensajes de error necesarios para localizar el fallo
- Falta definición funcional o regla de negocio para elegir entre opciones válidas
- Las respuestas del usuario siguen siendo ambiguas o contradictorias

Si hay bloqueo → **Modo C**.
Si hay confianza suficiente → **Paso 6**.

### Paso 6: Producir propuesta y generar archivo markdown

Seleccionar el modo de salida apropiado según el modo de análisis (funcional/técnico) y generar tanto la respuesta como el archivo markdown.

---

## Modos de salida

### Modo A-Funcional: Análisis funcional (contexto suficiente, sin código)

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

### Modo A-Técnico: Propuesta técnica (contexto suficiente, con código)

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

### Modo B: Contexto faltante, preguntas primero

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

### Modo C: Bloqueado tras investigación

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

### Modo D: Análisis funcional extendido + historial

Usar cuando el usuario pide "con contexto de HU anteriores", "análisis funcional extendido", o "dependencias CMM".

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

## Análisis de artefactos CMM

Cuando el usuario pega artefactos CMM (SOAP/XML, JSON, logs), aplicar este análisis específico:

### Request CMM
- Identificar la operación invocada (nombre del servicio, método)
- Listar campos enviados y sus valores
- Detectar campos vacíos, nulos, o con formato sospechoso
- Comparar contra la interfaz esperada si está disponible

### Response CMM
- Identificar si es éxito o error
- Si error: extraer código CMM (`cmmCode`), mensaje (`cmmMsg`), detalle (`detail`), source
- Analizar qué significa el error en contexto del flujo
- Si éxito: verificar que los campos de respuesta sean coherentes con lo esperado

### Request + Response juntos
- Trazar el flujo: qué se envió → qué se recibió → es coherente?
- Identificar discrepancias entre lo enviado y la respuesta
- Proponer causa probable del error o comportamiento inesperado

### Interfaz/Contrato CMM
- Mapear campos obligatorios vs opcionales
- Identificar tipos de datos y restricciones
- Comparar contra el request/response real si están disponibles

---

## Archivo de salida

**Siempre se genera un archivo markdown** con el análisis completo, independientemente del modo.

**Ruta del archivo:**
- Siempre → `~/Desktop/Análisis/<JIRA-KEY>-analisis.md` (NO generar archivos dentro del repositorio)

**Estructura del archivo:**

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

Crear el directorio de salida si no existe antes de escribir el archivo.

---

## Reglas

1. **Funcional primero** — el default es análisis funcional. Solo ir a código si el usuario lo pide o el ticket es claramente de implementación.
2. **Jamás inventar** — si el ticket es ambiguo o faltan requisitos, escalar por niveles. No inventar detalles.
3. **Escalamiento antes que preguntas** — investigar Jira, Confluence (y código si aplica) ANTES de preguntar al usuario.
4. **Artefactos del usuario son evidencia de primera clase** — logs, payloads, interfaces pegados en el chat tienen la misma validez que datos de Jira o código.
5. **Pedir artefactos CMM proactivamente** — si el problema involucra servicios CMM y no hay evidencia técnica, preguntar por trace number o pedir que peguen request/response.
6. **Etiquetar suposiciones** — toda suposición debe marcarse explícitamente como `[SUPOSICION]` o `[NO CONFIRMADO]`.
7. **Archivo siempre** — generar el archivo markdown de salida en todos los casos, sin excepción.
8. **Propuesta concisa** — 1-2 páginas salvo tickets muy complejos.
9. **Terminología del proyecto** — usar nombres de features, servicios y flujos tal como aparecen en Jira, Confluence y el código.
10. **Convenciones de código** (solo modo técnico) — branch naming `feature/JIRA-ID` o `fix/JIRA-ID`, commits en español.

## Referencias opcionales (NO se leen por defecto)

Estos archivos pueden leerse manualmente si se necesita contexto adicional:

- `docs/ai/feature-map.md` — mapa de features y sus rutas
- `docs/ai/context-index.json` — índice generado de contexto
- `docs/ai/examples/*.md` — 4 ejemplos de análisis
- `docs/ai/clarification-playbook.md` — playbook de preguntas (deprecado, integrado aquí)
- `docs/ai/analysis-template.md` — template de análisis (deprecado, integrado aquí)
- `docs/ai/history-heuristics.md` — heurísticas de búsqueda (deprecado, integrado aquí)
- `docs/ai/external-evidence-model.md` — modelo de evidencia externa (deprecado, integrado aquí)
