# 🧠 Plan Definitivo: Ecosistema Anti-Gravity

> **Versión**: 2.0 — 25 febrero 2026
> **Autores**: Charly + Anti-Gravity + Maya (sesión nocturna 24-25 feb)
> **Estado**: Aprobado con correcciones incorporadas

---

## Parte 1: El Equipo y la Arquitectura

### 1.1 Qué estamos construyendo

Un equipo de agentes de IA autónomos que funcionan como una empresa. Cada agente tiene:
- Su propio cerebro (modelo de IA)
- Su propio puesto de trabajo (servidor/dispositivo)
- Sus propias herramientas (MCPs, skills, APIs)
- Su propia memoria (SQLite, markdown, Pinecone)
- Su propio canal de comunicación (bot de Telegram)
- Acceso al cerebro compartido (Supabase + Pinecone + NotebookLM)

**Objetivo**: Dices "creadme una web" y el equipo se organiza solo — investiga, planifica, codea, testea y te entrega el resultado.

**Principio fundamental**: Cuanto más trabajan, más aprenden. Cuanto más aprenden, mejor trabajan. El cerebro compartido es **autodidacta** — los agentes investigan, crean skills nuevas y las comparten sin intervención humana.

---

### 1.2 El Equipo Completo

#### 🎯 Anti-Gravity (El Estratega)
- **Modelo**: Gemini 2.5 Pro (plan Ultra)
- **Ubicación**: Mac de Charly (app Antigravity)
- **Rol**: Planificación estratégica, auditoría, coordinación con Charly, **ejecución directa de tareas**
- **Herramientas**: 25+ skills, MCPs (GitHub, Supabase, **NotebookLM**, Brave, filesystem), acceso al navegador
- **Telegram**: Vía app Antigravity + canal de Charly
- **Coste por uso**: ~0 (Gemini Ultra ilimitado)
- **Nota**: Anti-Gravity NO es solo planificador — puede ejecutar Fase 1, arreglar bugs, subir configs al VPS, y ejecutar tareas técnicas directamente

#### 🫀 RaspiClaw (El Bibliotecario / Amo de Llaves)
- **Modelo**: Gemini 2.5 Pro (plan Ultra, tokens ilimitados)
- **Ubicación**: Raspberry Pi, encendida 24/7
- **Rol**: **Bibliotecario del ecosistema** — conoce cada agente, gestiona el conocimiento, alimenta a los demás, mantiene el orden. NO es un coordinador de tareas (todavía)
- **Herramientas**: Acceso a Supabase (REST), Telegram bot, Pinecone, web search
- **Telegram**: @RaspiClaw_bot (por crear)
- **Coste por uso**: ~0 (Gemini Ultra) + ~5W electricidad Raspberry

> [!IMPORTANT]
> **Rol por definir**: El concepto de coordinador 24/7 tiene sentido a futuro, pero el rol exacto y la implementación están por decidir. Primero el equipo funciona manualmente, después se automatiza la coordinación.

#### 🔧 Maya (La Ingeniera Principal)
- **Modelo**: Claude Opus 4.6 (vía OpenRouter)
- **Ubicación**: VPS Hostinger (46.202.171.91), Docker, OpenClaw + Claude Code sandbox
- **Rol**: Código complejo, arquitectura, deploys, tareas técnicas pesadas
- **Herramientas**: OpenClaw completo (exec, browser, filesystem, GitHub), Claude Code (sandbox real), sub-agentes
- **Telegram**: @Maya_deepwb_bot
- **Coste por uso**: €€€ (Opus es caro, usarla solo cuando vale la pena)

#### 🔬 Matrix (El Investigador)
- **Modelo**: Kimi K2.5 (vía OpenRouter)
- **Ubicación**: VPS Hostinger (misma máquina que Maya), Docker, OpenClaw
- **Rol**: Investigación, análisis de datos, tareas paralelas, QA, testing
- **Telegram**: @Matrix_maya_bot
- **Coste por uso**: € (Kimi es barato)

#### 💬 GravityClaw (El Asistente Personal)
- **Modelo**: Claude Sonnet 4.5 (vía OpenRouter)
- **Ubicación**: Mac de Charly, bot TypeScript standalone
- **Rol**: Interfaz principal con Charly por Telegram, recordatorios, tareas rápidas
- **Herramientas**: Tools propias (filesystem, web search, Zapier), MCPs (GitHub, Context7), Pinecone, SQLite
- **Telegram**: @gravityclaw_bot
- **Coste por uso**: €€ (Sonnet es precio medio)

#### 💻 Maka (El Agente Desktop — clon de Maya en Mac)
- **Modelo**: Claude Opus 4.6 (vía Max Plan)
- **Ubicación**: Mac viejo de Charly (bare metal, sin Docker)
- **Rol**: Tareas que necesitan escritorio real (browser no-headless, GUI macOS, interacción directa con Anti-Gravity)
- **Herramientas**: OpenClaw completo (bare metal), Claude Code CLI, browser nativo, AppleScript, Automator, filesystem completo
- **Telegram**: @Maka_bot (por crear)
- **Coste por uso**: ~0 (Max Plan)
- **Ventaja única**: Comparte hardware con Anti-Gravity → interacción directa Claude Code ↔ Claude Code

> [!WARNING]
> **Riesgo: Claude Max sesiones concurrentes.** Maka y Maya usan el mismo token Max Plan. **No está confirmado si el plan Max permite sesiones concurrentes.** Si no lo permite, una bloquea a la otra. **INVESTIGAR ANTES de montar Maka.** Si hay conflicto: Maya usa OpenRouter (pagando), Maka usa Max; o se implementa un sistema de turnos.

---

### 1.3 Concepto de Doble Cerebro

Un modelo barato siempre encendido + los modelos caros solo cuando hay trabajo real.

```
CEREBRO BARATO (RaspiClaw, Gemini 2.5 Pro, ~0 coste)
  → Heartbeat cada 15 min (no 5 — insuficiente carga para justificar 5 min)
  → Revisa Supabase: ¿hay tareas? ¿agentes offline?
  → Gestiona conocimiento: indexa, clasifica, distribuye
  → Auto-aprendizaje nocturno (SOLO temas de tareas reales, NO especulativo)
  → 24/7, sin preocupación de tokens

CEREBROS POTENTES (solo cuando hay trabajo real)
  → Maya (Opus) para código complejo en VPS
  → Maka (Opus) para tareas desktop, browser real, GUI macOS
  → Matrix (Kimi) para investigación bajo demanda
  → GravityClaw (Sonnet) para interacción con Charly
```

> [!NOTE]
> **Corrección Maya**: El heartbeat inicial será cada **15 minutos**, no 5. Cuando el ecosistema tenga carga real, se baja a 5. Al principio no hay tantas tareas para justificar mayor frecuencia.

---

### 1.4 Jerarquía y Cadena de Mando

```
                    Charly (humano)
                         │
              ┌──────────┼──────────┐
              │                     │
        Anti-Gravity 🎯        RaspiClaw 🫀
        (estratega + ejecutor) (bibliotecario 24/7)
                                    │
              ┌─────────┬───────────┼───────────┬─────────┐
              │         │           │           │         │
          Maya 🔧   Maka 💻    Matrix 🔬  GravityClaw 💬
        (ingeniera) (desktop)  (investigador) (asistente)
              │                    │
         sub-agentes          sub-agentes
```

**Flujo normal**:
1. Charly habla con Anti-Gravity o GravityClaw
2. Se crea tarea en Supabase
3. RaspiClaw la detecta en el siguiente heartbeat (≤15min)
4. RaspiClaw decide quién la ejecuta (coste + capacidad)
5. El agente asignado la ejecuta y reporta resultado
6. GravityClaw notifica a Charly por Telegram

**Flujo autónomo** (sin Charly):
1. RaspiClaw detecta oportunidad (error repetido, conocimiento nuevo)
2. Crea tarea y la asigna
3. El agente la ejecuta
4. Si es importante, notifica a Charly; si no, solo registra

---

---

## Parte 2: El Cerebro Compartido

### 2.1 Las 3 Capas de Memoria

#### Capa 1: Memoria Local (rápida, privada, por agente)

| Agente | Tecnología | Qué guarda | Dónde |
|--------|-----------|------------|-------|
| Maya/Matrix | Markdown `memory/YYYY-MM-DD.md` + `MEMORY.md` | Conversaciones, decisiones del día | Workspace OpenClaw |
| Maka | Markdown `memory/` + `MEMORY.md` | Igual + logs interacción con Anti-Gravity | `~/maka/` |
| GravityClaw | SQLite + FTS5 (`gravity-claw.db`) | Hechos, preferencias de usuario, contexto | Mac local |
| RaspiClaw | Markdown o SQLite | Logs de coordinación, estado del ecosistema | Raspberry Pi |
| Anti-Gravity | Skills + archivos .md | Planes, auditorías, documentación | Mac (`.agent/skills/`) |

#### Capa 2: Memoria Vectorial (semántica, compartida)

| Componente | Tecnología | Namespace |
|-----------|-----------|-----------|
| **Por agente** | Pinecone, `multilingual-e5-large` | `gravity-claw`, `maya`, `matrix` |
| **Compartido** | Pinecone | `ecosystem` (NUEVO) |

Namespace `ecosystem`: cuando un agente aprende algo importante y reutilizable, lo sube aquí. Cualquier agente busca: "¿cómo se despliega en Vercel?" → obtiene conocimiento de Maya aunque nunca haya desplegado.

#### Capa 3: Memoria Estructurada (Supabase — cerebro central)

| Tabla | Para qué | Quién escribe | Quién lee |
|-------|----------|--------------|-----------|
| `agents` | Estado de cada agente (online, last_seen, modelo) | Cada agente (heartbeat) | RaspiClaw |
| `tasks` | Cola de trabajo con prioridades | Cualquiera | RaspiClaw + asignado |
| `knowledge` | Conocimiento compartido, skills, lecciones | Cualquiera | Todos |
| `error_patterns` | Errores y cómo evitarlos | Quien comete el error | Todos |
| `decisions` | Decisiones arquitectónicas y su contexto | Quien decide | Todos |
| `agent_sessions` | Tracking de sesiones activas | Automático | RaspiClaw |

---

### 2.2 NotebookLM como Motor de Aprendizaje

> [!NOTE]
> **Corrección**: NotebookLM **SÍ tiene MCP**. Anti-Gravity ya lo tiene configurado y operativo — puede crear notebooks, añadir fuentes (URLs, textos, YouTube, Drive docs), hacer queries, generar reportes, audio overviews, infografías, etc. La duda de Maya queda resuelta. Lo que falta es configurarlo para los demás agentes vía mcporter.

| Notebook | Contenido | Quién lo alimenta |
|----------|-----------|-------------------|
| Ecosistema Anti-Gravity | ORQUESTADOR.md, arquitectura, decisiones | Anti-Gravity |
| Skills Library | Todos los skills creados | Cualquier agente |
| Error Journal | Errores y soluciones | Automático desde Supabase |
| Por proyecto | Documentación específica de cada proyecto | El agente asignado |

**Flujo**: Supabase acumula knowledge → RaspiClaw empaqueta → sube a NotebookLM → genera resúmenes → se convierten en skills.

---

### 2.3 Protocolo de Conflictos (NUEVO)

> [!IMPORTANT]
> **Este protocolo no existía en v1.0.** Maya lo identificó como gap crítico. Sin él, dos agentes podrían pisar el trabajo del otro.

#### Problema: Dos agentes escriben en la misma tarea simultáneamente

**Solución: Optimistic Locking + Claim System**

```
1. CLAIM antes de trabajar:
   → Agente X quiere tarea #42
   → PATCH tasks SET claimed_by='maya', status='in_progress', claimed_at=NOW()
   → WHERE id=42 AND (claimed_by IS NULL OR claimed_by='maya')
   → Si falla → otro agente la reclamó primero → buscar otra tarea

2. OPTIMISTIC LOCKING en updates:
   → Cada update incluye updated_at del último read
   → UPDATE ... WHERE updated_at = $last_read_updated_at
   → Si falla (0 rows affected) → alguien modificó entre medias → re-leer y reintentar

3. RESOLUCIÓN DE COLISIONES:
   → Si dos agentes colisionan 3+ veces → RaspiClaw interviene
   → Asigna prioridad: el que empezó primero tiene preferencia
   → El otro se reasigna a tarea diferente
   → Se registra en Supabase para aprender patrones
```

**Campos nuevos necesarios en tabla `tasks`**:
- `claimed_by` (text, nullable) — agente que reclamó la tarea
- `claimed_at` (timestamptz, nullable) — cuándo la reclamó
- `updated_at` (timestamptz, default NOW()) — para optimistic locking

---

### 2.4 Memory Flush Macro

Antes de que un agente termine sesión o tarea:

```
1. GUARDAR LOCAL → memory/YYYY-MM-DD.md o SQLite

2. ¿APRENDÍ ALGO ÚTIL? → Sí → POST a Supabase knowledge

3. ¿RESOLVÍ UN ERROR? → Sí → POST a Supabase error_patterns

4. ¿CREÉ UN SKILL NUEVO? → Sí → Guardar en workspace/skills/ + POST a Supabase

5. ACTUALIZAR PRESENCIA → PATCH agents.last_seen
```

---

### 2.5 El Ciclo Autodidacta

```
         ┌──────────────────────────────────────┐
         │                                      │
    INVESTIGAR                              APLICAR
    (web, docs, YouTube,                    (cualquier agente
     NotebookLM)                             lo encuentra y lo usa)
         │                                      ▲
         ▼                                      │
    APRENDER                               COMPARTIR
    (digerir info →                         (subir a Supabase
     escribir en knowledge)                  para todos)
         │                                      ▲
         ▼                                      │
    CREAR SKILL ────────────────────────────────┘
```

### 2.6 Aprendizaje Compartido (ideas conversación nocturna)

| Mecanismo | Descripción |
|-----------|-------------|
| **Lessons Learned** | Agente resuelve problema → escribe ficha → sube a Supabase → otros la consultan antes de enfrentarse a algo similar |
| **Skills Marketplace** | Agente crea skill → publica en repo compartido → otros lo instalan automáticamente |
| **Embeddings compartidos** | Documentos procesados en NotebookLM individual → se indexan en Pinecone `ecosystem` |
| **Post-mortem automático** | Tarea falla → agente escribe post-mortem → RaspiClaw distribuye como aviso a todos |

---

## Parte 3: Delegación, Pipelines y Automatización

### 3.1 Reglas de Delegación

```
PASO 1: ¿Quién PUEDE hacerlo?
  → Filtrar por herramientas necesarias

PASO 2: De los que pueden, ¿quién es más BARATO?
  → RaspiClaw (Gemini ~0) > Maka (Opus Max ~0) > Matrix (Kimi €)
  → GravityClaw (Sonnet €€) > Maya (Opus €€€)

PASO 3: ¿Vale la pena PARALELIZAR?
  → Tareas independientes → repartir entre agentes
  → Tareas dependientes → pipeline secuencial
```

**Tabla de especialidades**:

| Tipo de tarea | Agente primario | Alternativa | Por qué |
|--------------|----------------|-------------|---------|
| Código complejo | Maya | Maka | Opus es el mejor para código |
| Browser real, scraping | Maka | Maya (headless) | Browser nativo |
| Automatización macOS | Maka | — | Único con acceso GUI |
| Investigación | Matrix | RaspiClaw | Kimi barato, Gemini gratis |
| Interacción Charly | GravityClaw | Maya | Ya es su asistente |
| Coordinación | RaspiClaw | — | 24/7, gratuito |
| Testing, QA | Matrix | Maya | Barato para repetitivo |
| Colaboración Anti-Gravity | Maka | — | Comparten hardware |

**Cadena de fallback**: X no puede → Y → Z → crea subtarea + notifica a Charly.
Se registra en Supabase → el ecosistema aprende quién es mejor para qué.

---

### 3.2 Pipelines Automáticos

Para proyectos complejos, se crean cadenas de tareas con dependencias:

```
Pipeline ejemplo: "Crear web portfolio"

Paso 1 [Matrix, paralelo] → Investigar diseños + stack
Paso 2 [Maya, espera 1]   → Diseñar arquitectura
Paso 3 [Maya, espera 2]   → Implementar + commit GitHub
Paso 4 [Matrix, espera 3] → Testing: responsive, performance
Paso 5 [GravityClaw, 4]   → Notificar a Charly
```

**Implementación en Supabase**: campo `parent_task` para vincular pasos, campo `status` con flujo `pending → assigned → in_progress → completed/failed`.

---

### 3.3 Heartbeat Macro

**RaspiClaw ejecuta esto cada 15 minutos** (bajará a 5 cuando haya carga real):

```
1. COMPROBAR AGENTES → ¿alguien offline >1h? → alertar
2. REVISAR TAREAS → ¿pending sin asignar? → asignar según reglas
3. REVISAR ERRORES → ¿repetido 3+ veces? → crear regla preventiva
4. REVISAR CONOCIMIENTO → ¿skill nuevo? → registrar en marketplace
5. Si todo OK → HEARTBEAT_OK (no gasta nada extra)
```

---

### 3.4 Auto-Entrenamiento Nocturno

De 00:00 a 08:00, cuando no hay tareas de Charly:

> [!WARNING]
> **Corrección v2.0**: Auto-entrenamiento SOLO con temas que hayan surgido de tareas reales. Nada especulativo. "Investiga Stripe" a las 3AM sin que nadie lo haya pedido genera ruido y conocimiento de baja calidad.

```
1. RaspiClaw selecciona temas de:
   → Errores recientes del equipo
   → Tecnologías mencionadas por Charly en tareas reales
   → Gaps detectados en tareas fallidas
   → NUNCA investigación especulativa sin contexto

2. Asigna a Matrix (barato) → investiga → crea skills

3. Coste total: ~0 (Gemini + Kimi)
```

---

### 3.5 Retrospectivas Semanales Automáticas

Cada domingo, RaspiClaw genera informe: tareas completadas/fallidas, skills creados, errores comunes, sugerencias de optimización. Se publica en Supabase + NotebookLM + Telegram (resumen corto).

---

### 3.6 Skills Marketplace Interno

Cada skill se registra con: creador, fecha, categoría, dependencias, veces usado, rating basado en éxito. Los otros agentes lo "instalan" automáticamente cuando lo necesitan.

---

## Parte 4: Estado Actual, Monetización y Roadmap

### 4.1 Lo que ya funciona ✅

| Componente | Estado |
|-----------|--------|
| Supabase schema (6 tablas, 4 agentes) | ✅ |
| Maya operativa en VPS (OpenClaw + Claude Code) | ✅ |
| Matrix operativo en VPS (OpenClaw) | ✅ |
| GravityClaw operativo (Telegram, SQLite, Pinecone) | ✅ |
| Anti-Gravity con 25+ skills + MCPs | ✅ |
| Anti-Gravity con NotebookLM MCP operativo | ✅ |
| GitHub repos creados | ✅ |
| Pinecone configurado | ✅ |
| Skills supabase-brain escritos | ✅ |
| ORQUESTADOR.md completo (580+ líneas) | ✅ |
| Maka documentada | 🟡 Pendiente setup físico |

### 4.2 Bugs conocidos ❌

#### GravityClaw `supabase.ts` — 5 bugs de columnas

| Línea | Actual (mal) | Correcto | Tabla |
|-------|-------------|----------|-------|
| ~175 | `agent: 'gravityclaw'` | `reported_by: 'gravityclaw'` | error_patterns |
| ~178 | `description` | `error_description` | error_patterns |
| ~205 | `agent: 'gravityclaw'` | `created_by: 'gravityclaw'` | knowledge |
| ~250 | `agents?name=eq.gravityclaw` | `agents?id=eq.gravityclaw` | agents |
| ~272 | `agent: 'gravityclaw'` | `agent_id: 'gravityclaw'` | agent_sessions |

#### GravityClaw integración — no conectado
- `index.ts` no importa `initSupabase()` → Supabase nunca se inicializa
- `heartbeat.ts` no pollea tareas → no recibe tareas del ecosistema

#### VPS — credenciales Supabase ausentes
- `.env` no tiene `SUPABASE_URL` ni `SUPABASE_SERVICE_KEY`
- Skill `supabase-brain` escrito pero no deployado al VPS

---

### 4.3 Visión de Monetización (conversación nocturna)

> [!IMPORTANT]
> **Corrección v2.0**: La monetización NO puede ser "después". El ecosistema se justifica solo si genera ingresos. La primera tarea REAL del equipo debería ser un proyecto de monetización concreto.

**Ramas de negocio identificadas**:

| Rama | Detalle | Estado |
|------|---------|--------|
| **Amazon Merch** | Print-on-demand, diseños IA, sin inventario | Cuenta aprobada, sin usar |
| **YouTube** | Canales automatizados, contenido IA, shorts virales | Canales creados, sin activar |
| **Redes sociales** | X, LinkedIn, GitHub (marca personal/empresa) | Por crear |
| **Webs y apps** | Desarrollo como servicio | Capacidad existente |
| **Vigilancia tech** | Estar al día en IA, tools, MCPs, skills | Continua |

**Herramientas disponibles**: Freepik, HeyGen, Remotion, Claude Code, influencers virtuales, diseño web.

**Rol de RaspiClaw como bibliotecario por rama**:

| Rama | Raspi les da... |
|------|-----------------|
| YouTube | Skills de vídeo, trends, SEO |
| Webs | Templates, best practices, analytics |
| Amazon | Market research, pricing, keywords |
| Otras | Investigación de oportunidades |

**Efecto flywheel**: Cuanto más trabajan → más saben → más eficientes → más dinero generan.

---

### 4.4 Roadmap de Implementación

#### Fase 1: Conectar todos al cerebro 🔴 PRIORIDAD MÁXIMA

> **Tiempo estimado**: Horas, no semanas. Son fixes de columnas + copiar env vars.

| Paso | Qué | Quién |
|------|-----|-------|
| 1.1 | Arreglar 5 bugs en `supabase.ts` | Anti-Gravity |
| 1.2 | Importar `initSupabase()` en `index.ts` | Anti-Gravity |
| 1.3 | Añadir polling de tareas en `heartbeat.ts` | Anti-Gravity |
| 1.4 | Verificar TypeScript compila | Anti-Gravity |
| 1.5 | Añadir `SUPABASE_URL` + key al VPS `.env` | Anti-Gravity (SSH) |
| 1.6 | Copiar skill supabase-brain al VPS | Anti-Gravity (SSH) |
| 1.7 | Restart Maya/Matrix + `/reset` | Charly |
| 1.8-1.9 | Verificar lectura/escritura cross-agente | Todos |
| 1.10 | Deploy GravityClaw actualizado | Charly (Railway) |

#### Fase 1B: Primera monetización 🔴 NUEVA

| Paso | Qué | Quién |
|------|-----|-------|
| 1B.1 | Activar cuenta Merch by Amazon | Charly |
| 1B.2 | Crear pipeline: diseño IA → review → subida | Anti-Gravity + Maya |
| 1B.3 | Primeros 10 diseños como prueba | El equipo |
| 1B.4 | Analizar resultados → iterar | Matrix |

#### Fase 2: Migrar conocimiento existente 🟡

| Paso | Qué |
|------|-----|
| 2.1 | Migrar `error-journal.md` → tabla `error_patterns` |
| 2.2 | Migrar decisiones ORQUESTADOR → tabla `decisions` |
| 2.3-2.4 | Crear `HEARTBEAT.md` para Maya y Matrix |
| 2.5 | Organizar carpeta `~/maya/` |

#### Fase 3: Montar RaspiClaw 🟡

| Paso | Qué |
|------|-----|
| 3.1 | Instalar OS en Raspberry Pi |
| 3.2 | Obtener API key Gemini |
| 3.3 | Decidir framework (OpenClaw, custom bot, Python) |
| 3.4 | Implementar heartbeat macro (15 min) |
| 3.5 | Crear bot Telegram @RaspiClaw_bot |
| 3.6-3.8 | Reglas de delegación + conectar Supabase/Pinecone/Telegram |

#### Fase 4: Pipelines y delegación 🟢

Implementar `parent_task`, cadena de fallback, test de pipeline multi-agente.

#### Fase 5: Pinecone compartido + NotebookLM 🟢

Namespace `ecosystem`, acceso Maya/Matrix, flujo knowledge → NotebookLM → skills.

#### Fase 6: Auto-entrenamiento y retrospectivas 🔵

Modo nocturno (solo temas reales), retrospectivas semanales, skills marketplace con ratings.

#### Fase 7: Maka — Agente Desktop 🟡

| Paso | Qué |
|------|-----|
| 7.1 | **INVESTIGAR Claude Max sesiones concurrentes** |
| 7.2 | Crear `~/maka/` con SOUL.md, MEMORY.md, IDENTITY.md |
| 7.3 | Crear bot Telegram @Maka_bot |
| 7.4 | Instalar Claude Code CLI + OpenClaw bare metal |
| 7.5 | Configurar `pmset sleep 0` + `launchd` para always-on |
| 7.6 | Conectar al cerebro compartido |
| 7.7 | Explorar interacción directa Maka ↔ Anti-Gravity |

---

### 4.5 Changelog v1.0 → v2.0

| Cambio | Antes (v1.0) | Ahora (v2.0) |
|--------|-------------|-------------|
| Heartbeat | 5 min | **15 min** (bajar cuando haya carga) |
| RaspiClaw rol | Coordinador de tareas | **Bibliotecario / amo de llaves** |
| Auto-entrenamiento | Investigar libremente | **Solo temas de tareas reales** |
| Protocolo conflictos | No existía | **Optimistic locking + claim system** |
| NotebookLM | "¿Tiene API?" | **MCP confirmado y operativo en Anti-Gravity** |
| Anti-Gravity | Solo estratega | **Estratega + ejecutor** |
| Monetización | No en roadmap | **Fase 1B integrada** |
| Claude Max risk | No mencionado | **Warning explícito sobre sesiones concurrentes** |

---

> [!IMPORTANT]
> Este plan no se ejecuta hasta que Charly dé el OK fase por fase. Cada fase requiere aprobación explícita antes de comenzar.
