# 🧠 Anti-Gravity Orchestrator — Estado del Ecosistema

> **Última actualización**: 2026-02-21 11:36
> **Actualizado por**: Anti-Gravity (GitHub sync: ambos repos sincronizados, workflow creado, sistema redacción)

Este documento es la **fuente de verdad central** del ecosistema Anti-Gravity + Maya.
Cualquier sesión de cualquier modelo de IA debe leer este documento antes de hacer cambios.

---

## 1. Identidad del Ecosistema

| Componente | Rol | Dónde vive |
|-----------|-----|------------|
| **Charly** | Owner, humano, toma decisiones finales | — |
| **Anti-Gravity** | Cerebro orquestador, investigación, planificación | Mac local (Antigravity IDE) |
| **Maya** | Agente 24/7, orquestadora, asistente personal | VPS Hostinger (Docker) — agente `main` |
| **Matrix** | Brazo ejecutor, tareas delegadas por Maya | VPS Hostinger (Docker) — agente `worker` |
| **n8n** | Automatización de workflows | VPS Hostinger (Docker) |

### Relación Anti-Gravity ↔ Maya ↔ Matrix
- Anti-Gravity investiga, planifica y configura
- Maya orquesta, supervisa y está disponible 24/7 via Telegram
- Matrix ejecuta tareas delegadas por Maya (modelo: Kimi K2.5 via OpenRouter)
- Comunicación Maya↔Matrix via `tools.agentToAgent` + `sessions_spawn`
- Comunicación asíncrona Anti-Gravity↔Maya via directorio handoff en VPS

---

## 2. Infraestructura VPS

| Dato | Valor |
|------|-------|
| **IP** | `46.202.171.91` |
| **Proveedor** | Hostinger |
| **SSH** | `ssh root@46.202.171.91` (key-only, password disabled) |
| **SSH Key** | `~/.ssh/id_ed25519` |
| **OS** | Ubuntu (Docker host) |

### Puertos y Firewall (UFW)

| Puerto | Servicio | Estado |
|--------|----------|--------|
| 22 | SSH | ✅ Abierto (key-only) |
| 80 | HTTP (Traefik) | ✅ Abierto |
| 443 | HTTPS (Traefik) | ✅ Abierto |
| 40362 | OpenClaw Dashboard | 🔒 Solo localhost |
| 5432 | PostgreSQL | 🔒 Bloqueado |
| 3456 | Claude Max Proxy | 🔒 Solo `172.18.0.0/24` (bridge Docker) |

### HTTPS / Traefik
- **Dominio**: `openclaw.technovedades.com`
- **DNS**: A record → `46.202.171.91`
- **Certificado**: Let's Encrypt (auto-renewal)
- **TLS**: TLSv1.3

### Docker
- **Contenedor Maya**: `openclaw-pyi4-openclaw-1`
- **Redes**: `openclaw-pyi4_default`, `n8n-mcp_default`
- **Traefik**: `n8n-mcp-traefik-1`

---

## 3. Maya (OpenClaw)

### Archivos clave en VPS

| Archivo | Ruta |
|---------|------|
| Docker Compose | `/docker/openclaw-pyi4/docker-compose.yml` |
| Config | `/docker/openclaw-pyi4/data/.openclaw/openclaw.json` |
| SOUL.md (Maya) | `/docker/openclaw-pyi4/data/.openclaw/workspace/SOUL.md` |
| SOUL.md (Matrix) | `/docker/openclaw-pyi4/data/.openclaw/workspace-worker/SOUL.md` |
| HEARTBEAT.md | `/docker/openclaw-pyi4/data/.openclaw/workspace/HEARTBEAT.md` |
| Skills | `/docker/openclaw-pyi4/data/.openclaw/workspace/skills/` |
| Memory | `/docker/openclaw-pyi4/data/.openclaw/workspace/memory/` |
| MCP Config | `/data/.mcporter/mcporter.json` (dentro del container) |
| Env vars | `/docker/openclaw-pyi4/.env` |
| Email env | `/docker/openclaw-pyi4/.env.email` |

### Modelo Actual
- **Primary**: `claude-max/claude-opus-4-6` ← **Claude Opus 4.6 via Max Plan (sin coste API)**
- **Versión OpenClaw**: `2026.2.17` (actualizado 2026-02-19)
- **Config reload**: `hybrid` (hot-reload automático)

### Claude Max API Proxy (NUEVO — 2026-02-18)

Maya usa Claude Opus 4.6 gratis via el plan Max de Charly, a través de un proxy local en el VPS.

| Componente | Detalle |
|-----------|---------|
| **Servicio systemd** | `claude-proxy.service` (arranca con VPS, auto-restart) |
| **Usuario sistema** | `claude-proxy` (sin shell, sin privilegios) |
| **Token OAuth** | `/etc/claude-proxy/env` (chmod 600, válido 1 año) |
| **Código proxy** | `/opt/claude-proxy/app` (compilado desde source) |
| **WorkingDirectory** | `/docker/openclaw-pyi4/data/.openclaw/workspace` (sandbox = workspace Maya) |
| **Escucha** | `172.18.0.1:3456` (solo bridge Docker, NO internet) |
| **UFW** | Solo permite `172.18.0.0/24 → :3456` |
| **Endpoint** | `http://172.18.0.1:3456/v1` (compatible OpenAI) |
| **Claude Code permisos** | `/opt/claude-proxy/.claude/settings.json` — Bash/Read/Write/Edit(*) permitidos |
| **ExecStartPre** | `+/bin/chmod o+rx` en `.openclaw` y `workspace` — fija permisos que Docker resetea |
| **Git safe.directory** | `/docker/openclaw-pyi4/data/.openclaw/workspace` (en `~/.gitconfig` del user) |
| **Grupo docker** | `claude-proxy` está en el grupo `docker` → puede usar `docker exec` sin sudo |
| **Sudoers** | `/etc/sudoers.d/claude-proxy-ocl` — (legacy, ya no necesario para docker exec) |
| **ocl-tool** | `/usr/local/bin/ocl-tool` — wrapper que llama al gateway interno (18789) via `docker exec` (sin sudo) |

#### Gestión del servicio proxy
```bash
systemctl status claude-proxy       # Estado
systemctl restart claude-proxy      # Reiniciar (auto-fix permisos via ExecStartPre)
journalctl -u claude-proxy -f       # Logs en tiempo real
curl http://172.18.0.1:3456/health  # Health check desde host
```

#### ⚠️ Procedimiento correcto de restart completo
```bash
# SIEMPRE en este orden:
cd /docker/openclaw-pyi4 && docker compose restart openclaw
systemctl stop claude-proxy && sleep 2 && systemctl start claude-proxy
```
> **IMPORTANTE**: Usar `stop + start`, NUNCA `restart`. El `restart` puede dejar procesos zombie que causan `code -13`.
> Docker restart resetea `chmod` de `.openclaw/` a `drwx------`. El `ExecStartPre` lo re-aplica automáticamente en el `start`.

#### Renovar token OAuth (caduca en 1 año)
1. En Mac local: ejecutar `claude auth token` → copiar nuevo token
2. En VPS: `nano /etc/claude-proxy/env` → reemplazar `CLAUDE_CODE_OAUTH_TOKEN`
3. `systemctl restart claude-proxy`

### Modelos Disponibles (16)

| ID | Alias | Uso |
|----|-------|-----|
| `claude-max/claude-opus-4-6` | Claude Opus 4.6 MAX | **Primary (plan Max, gratis)** |
| `claude-max/claude-opus-4` | Claude Opus 4 MAX | Disponible via Max |
| `claude-max/claude-sonnet-4` | Claude Sonnet 4 MAX | Disponible via Max |
| `claude-max/claude-haiku-4` | Claude Haiku 4 MAX | Disponible via Max |
| `openrouter/moonshotai/kimi-k2.5` | Kimi 2.5 | Fallback gratuito |
| `anthropic/claude-sonnet-4-5` | Claude Sonnet 4.5 | Coder agent |
| `google/gemini-3-flash-preview` | Gemini 3 Flash | Researcher agent |
| `openai/gpt-5.2` | ChatGPT 5.2 | Writer agent |
| `anthropic/claude-opus-4-1` | Claude Opus 4.1 | Architect agent |
| `anthropic/claude-opus-4` | Claude Opus 4 | Disponible |
| `anthropic/claude-opus-4-6` | Claude Opus 4.6 | Disponible |
| `openrouter/anthropic/claude-opus-4` | OR Claude Opus 4 | Disponible |
| `openrouter/anthropic/claude-sonnet-4` | OR Claude Sonnet 4 | Disponible |
| `openrouter/openai/gpt-4.5-turbo` | OR GPT-4.5 Turbo | Disponible |
| `openrouter/google/gemini-pro-2.0-exp` | OR Gemini Pro 2.0 | Disponible |
| `xai/grok-4-1-fast-reasoning` | Grok 4.1 Fast | Disponible |

### Agentes Multi-Agent (2) + Sub-agentes (5)

**Agentes principales** (aislados, workspace propio, Telegram propio):

| ID | Nombre | Modelo | Telegram |
|----|--------|--------|----------|
| `main` | Maya - Asistente Personal | `claude-max/claude-opus-4-6` | `@Maya_Matrix_bot` (default) |
| `worker` | Matrix - Brazo Ejecutor | `openrouter/moonshotai/kimi-k2.5` | `@Matrix_maya_bot` (worker) |

**Sub-agentes de Maya** (comparten workspace de Maya):

| ID | Nombre | Modelo |
|----|--------|--------|
| `coder` | Experto en Código | claude-sonnet-4.5 |
| `researcher` | Investigación y Análisis | gemini-3-flash-preview |
| `writer` | Redactor de Contenido | gpt-5.2 |
| `architect` | Arquitecto de Sistemas | claude-opus-4.1 |

### Multi-Agent Config
- **Bindings**: `main` → `telegram:default`, `worker` → `telegram:worker`
- **agentToAgent**: `enabled: true`, `allow: [main, worker]`
- **Maya subagents.allowAgents**: `[worker]` (puede spawnear a Matrix)
- **Worker subagents.allowAgents**: `[main]` (Matrix puede reportar a Maya)

### Heartbeat
- **Intervalo**: `60m`
- **Modelo**: `openrouter/moonshotai/kimi-k2.5` (gratuito — separado del cerebro principal)
- **Archivo de tareas**: `HEARTBEAT.md`
- **Nota**: El campo `heartbeat.model` fue aplicado vía hot-reload (v2026.2.12 confirmado OK)


### Tools Habilitados
- **Permitidos (21)**: read, write, edit, apply_patch, exec, process, web_search, web_fetch, browser, image, memory_search, memory_get, sessions_list, sessions_history, sessions_send, sessions_spawn, session_status, message, cron, gateway, agents_list
- **Denegados (4)**: nodes, canvas, llm_task, lobster
- **Exec security**: `allowlist` mode — safeBins: `curl`, `python3`, `node`
- **Browser**: `headless: true`, `noSandbox: true`, profile `openclaw`, Chromium 144, CDP port 18800

### Skills Bundled Activos (10 de 53)
`gog`, `github`, `tmux`, `session-logs`, `weather`, `summarize`, `clawhub`, `healthcheck`, `skill-creator`, `mcporter`

### Skills Instalados en VPS (5 custom)

| Skill | Función | Estado |
|-------|---------|--------|
| `google-oauth` | OAuth 2.0 para Gmail, Drive, Calendar | ✅ Activo |
| `himalaya-gmail-setup` | Setup de Himalaya para email CLI | ✅ Instalado |
| `n8n-workflow-automation` | Generador de workflows n8n | ✅ Instalado |
| `n8n-control` | Control directo de n8n via REST API | ✅ Instalado |
| `sqlite-storage` | Almacenamiento SQLite | ✅ Instalado |
| `school-report` | Informe diario colegio Carlos (3ºA, La Salle Paterna) | ✅ Instalado (Maya + Matrix) |

### MCP Servers (via mcporter)

| Servidor | Tools | Descripción |
|----------|-------|-------------|
| `filesystem` | 14 | read/write/edit files, directories, search |
| `memory` | 9 | Knowledge graph (entities, relations, observations) |

> Config: `/data/.mcporter/mcporter.json` — Maya usa `mcporter list/call` para interactuar.
> Fetch no incluido (Maya ya tiene `web_fetch` built-in).

### Heartbeat (Proactive Mode)

- **Intervalo**: `60m`
- **Archivo de tareas**: `HEARTBEAT.md`
- **Tareas**: Revisar handoff de Anti-Gravity, procesar memory, emails urgentes, cron jobs

---

## 4. Telegram (Multi-Bot)

### Bot Maya (default)
| Dato | Valor |
|------|-------|
| **Bot** | `@Maya_Matrix_bot` |
| **Bot Token** | `[REDACTED]` |
| **Account ID** | `default` |
| **DM Policy** | `pairing` |

### Bot Matrix
| Dato | Valor |
|------|-------|
| **Bot** | `@Matrix_maya_bot` |
| **Bot Token** | `[REDACTED]` |
| **Account ID** | `worker` |
| **DM Policy** | `allowlist` |

### Compartido
| Dato | Valor |
|------|-------|
| **User ID (Charly)** | `[REDACTED]` |
| **Teléfono** | `[REDACTED]` |
| **Stream mode** | `partial` |
| **Group Policy** | `allowlist` |

### Comandos Telegram
- `/start` — Iniciar
- `/help` — Ayuda
- `/reset` — Nueva sesión (aplica cambios en skills/SOUL.md/memory)
- `/settings` — Ajustes
- `/model` — Ver/cambiar modelo

---

## 5. Google OAuth 2.0

| Dato | Valor |
|------|-------|
| **Email** | `deepwbmayamatrix@gmail.com` |
| **Proyecto GCP** | `matrix-487518` |
| **Client ID** | `[REDACTED]` |
| **Client Secret** | En `~/maya/.maya-credentials.json` (local) |
| **APIs** | Gmail, Drive, Calendar, Sheets, YouTube (Data+Analytics+Reporting), Cloud Vision, Dialogflow, Workspace Marketplace |
| **Tokens VPS** | `/docker/openclaw-pyi4/data/.openclaw/workspace/skills/google-oauth/config/tokens.json` |
| **Estado** | ✅ Tokens activos y operativos (confirmado 2026-02-17) |
| **Nota** | Cuenta anterior (`maya.matrix102@gmail.com`) BLOQUEADA — no usar |

---

## 6. Credenciales (ubicaciones, sin valores en claro)

| Credencial | Ubicación |
|-----------|-----------|
| OpenRouter API Key | `openclaw.json` → `models.providers.openrouter.apiKey` |
| Anthropic API Key | `/docker/openclaw-pyi4/.env` → `ANTHROPIC_API_KEY` |
| OpenAI API Key | `/docker/openclaw-pyi4/.env` → `OPENAI_API_KEY` |
| Gemini API Key | `/docker/openclaw-pyi4/.env` → `GEMINI_API_KEY` |
| XAI API Key | `/docker/openclaw-pyi4/.env` → `XAI_API_KEY` |
| Telegram Bot Token | `openclaw.json` → `channels.telegram.botToken` |
| Gateway Token | `openclaw.json` → `hooks.token` |
| Google OAuth | VPS: `skills/google-oauth/config/` + Local: `~/maya/.maya-credentials.json` |
| Email (Maya) | `/docker/openclaw-pyi4/.env.email` → `MAYA_EMAIL` |

---

## 7. Skills Anti-Gravity (Mac local)

| Skill | Archivo | Función |
|-------|---------|---------|
| `maya-openclaw` | `~/.agent/skills/maya-openclaw/SKILL.md` | Comandos SSH, diagnostics, config Maya |
| `openclaw-reference` | `~/.agent/skills/openclaw-reference/SKILL.md` | Referencia completa OpenClaw |
| `orchestrator` | `~/.agent/skills/orchestrator/SKILL.md` | **ESTE** — Agente orquestador |
| `antigravity-skill-creator` | `~/.agent/skills/antigravity-skill-creator/SKILL.md` | Crear skills Anti-Gravity |
| `brainstorming` | `~/.agent/skills/brainstorming/SKILL.md` | Brainstorming |
| `brand-identity` | `~/.agent/skills/brand-identity/SKILL.md` | Identidad de marca |
| `error-handling-patterns` | `~/.agent/skills/error-handling-patterns/SKILL.md` | Patrones de manejo de errores |
| `reverse-engineering-videos` | `~/.agent/skills/reverse-engineering-videos/SKILL.md` | Ingeniería inversa de vídeos |
| `writing-plans` | `~/.agent/skills/writing-plans/SKILL.md` | Planes de escritura |

### Workflows Anti-Gravity

| Workflow | Archivo | Función |
|----------|---------|---------| 
| `/github-sync` | `~/maya/.agent/workflows/github-sync.md` | Sincronizar docs con GitHub (ambos repos) |
| `/learn-from-errors` | `~/maya/.agent/workflows/learn-from-errors.md` | Consultar error-journal antes de tareas complejas |
| `/log-error` | `~/maya/.agent/workflows/log-error.md` | Registrar error/lección en error-journal |

---

## 8. Handoff (Comunicación Asíncrona)

| Directorio | Función |
|-----------|---------|
| `/docker/openclaw-pyi4/data/handoff/from_antigravity/tasks/` | Anti-Gravity → Maya (tareas JSON) |
| `/docker/openclaw-pyi4/data/handoff/from_maya/results/` | Maya → Anti-Gravity (resultados) |
| `/docker/openclaw-pyi4/data/handoff/shared/knowledge_base.md` | Conocimiento compartido |


---

## 9. Reglas de Operación

### Comportamiento de Recarga

| Qué cambias | Auto-aplica? | Acción necesaria |
|------------|:---:|----------------|
| `openclaw.json` (config) | ✅ | Nada (hot-reload hybrid) |
| `SOUL.md` / `identity.md` | ❌ | `/reset` en Telegram |
| Skills (archivos `.md`) | ❌ | `/reset` en Telegram |
| `user.md` / `memory/` | ❌ | `/reset` en Telegram |
| `.env` / `.env.email` | ❌ | `docker compose restart` en VPS |

### Memoria
- **Persistente**: `workspace/memory/` — sobrevive a `/reset`
- **Sesión**: Chat messages — se borra con `/reset`
- **Orden de carga**: SOUL.md → user.md → memory/ → skills snapshot

### SOUL.md Actual
Maya tiene personalidad calibrada: respuestas directas, brevedad, humor natural, swearing cuando encaja. Sin formalidades. Warm competence.

---

## 10. Changelog

| Fecha | Cambio | Dónde | Estado |
|-------|--------|-------|--------|
| 2026-02-12 | Setup inicial Maya en VPS | VPS | ✅ |
| 2026-02-12 | Configurado Telegram bot | VPS | ✅ |
| 2026-02-13 | Configurado sub-agentes (5) con modelos especializados | openclaw.json | ✅ |
| 2026-02-13 | Instalado skill n8n-workflow-automation | VPS skills | ✅ |
| 2026-02-13 | Fix crash por Moonshot provider sin API key | openclaw.json | ✅ |
| 2026-02-13 | Cambiado default a kimi-k2.5 (gratuito) | openclaw.json | ✅ |
| 2026-02-13 | Configurado HTTPS via Traefik | VPS | ✅ |
| 2026-02-13 | Firewall UFW: solo 22, 80, 443 | VPS | ✅ |
| 2026-02-13 | SSH key-only auth habilitada | VPS | ✅ |
| 2026-02-13 | Fix identidad Maya (respondía como Charly) | SOUL.md | ✅ |
| 2026-02-14 | Google OAuth 2.0 configurado (Gmail, Drive, Calendar) | VPS + local | ✅ |
| 2026-02-14 | Skills google-oauth + himalaya-gmail instalados | VPS skills | ✅ |
| 2026-02-14 | Skill maya-openclaw actualizado con OAuth | Anti-Gravity | ✅ |
| 2026-02-14 | **Creado sistema Orquestador** (este documento + skill + workflow) | Anti-Gravity | ✅ |
| 2026-02-14 | App OAuth publicada en GCP (tokens permanentes) | Google Cloud | ✅ |
| 2026-02-14 | Heartbeat activado (`every: 60m`) + HEARTBEAT.md creado | openclaw.json + VPS | ✅ |
| 2026-02-14 | MCP servers configurados via mcporter (filesystem + memory) | VPS | ✅ |
| 2026-02-14 | Skill n8n-control creado (control de n8n via exec+curl) | VPS skills | ✅ |
| 2026-02-14 | mcporter v0.7.3 instalado (skill bundled activado) | VPS | ✅ |
| 2026-02-17 | OAuth tokens generados y verificados activos | VPS + GCP | ✅ |
| 2026-02-17 | Fix crash Maya (TLS undici `setSession` null bug) — restart | VPS | ✅ |
| 2026-02-17 | OpenClaw actualizado `v2026.2.9` → `v2026.2.12` (docker pull) | VPS | ✅ |
| 2026-02-17 | Fix crash loop Maya: browser tool roto por procesos Chromium huérfanos + locks estancados | VPS | ✅ |
| 2026-02-17 | Limpiados session locks + browser singleton locks tras crash | VPS | ✅ |
| 2026-02-17 | Verificado browser tool E2E: open, tabs, snapshot funcionando | VPS | ✅ |
| 2026-02-18 | Claude Code CLI instalado en VPS host (`/usr/bin/claude` v2.1.45) | VPS | ✅ |
| 2026-02-18 | claude-max-api-proxy instalado y compilado en `/opt/claude-proxy/app` | VPS | ✅ |
| 2026-02-18 | Servicio systemd `claude-proxy` configurado con hardening de seguridad | VPS | ✅ |
| 2026-02-18 | UFW: puerto 3456 solo para bridge Docker `172.18.0.0/24` | VPS | ✅ |
| 2026-02-18 | Provider `claude-max` añadido a openclaw.json (proxy en `172.18.0.1:3456`) | openclaw.json | ✅ |
| 2026-02-18 | **Primary model cambiado a `claude-max/claude-opus-4-6`** (Max plan, sin coste API) | openclaw.json | ✅ |
| 2026-02-18 | Main agent fijado a `claude-max/claude-opus-4-6` (override de Kimi corregido) | openclaw.json | ✅ |
| 2026-02-18 | Heartbeat configurado con `openrouter/moonshotai/kimi-k2.5` (gratuito, separado del cerebro) | openclaw.json | ✅ |
| 2026-02-18 | **Bug fix crítico: `[object Object]` en respuestas Maya** — `openai-to-cli.js` asumía `msg.content` como string, pero OpenClaw manda arrays; añadida función `extractText()` para manejar ambos formatos | VPS `/opt/claude-proxy/app/dist/adapter/openai-to-cli.js` | ✅ |
| 2026-02-18 | Archivadas 26 sesiones `.jsonl` corruptas en `agents/main/sessions/` (artefacto del restart) | VPS | ✅ |
| 2026-02-18 | **Proxy cwd cambiado** a `/docker/openclaw-pyi4/data/.openclaw/workspace` — Maya ya tiene acceso completo al workspace via Claude Code CLI sandbox. ExecStart cambiado a path absoluto. Permisos `o+rx` en `/docker/openclaw-pyi4/data/.openclaw` | VPS systemd | ✅ |
| 2026-02-18 | **Claude Code permisos desbloqueados** — `settings.json` con `Bash(*)/Read(*)/Write(*)/Edit(*)` + git safe.directory configurado. Maya opera sin pedir permisos | VPS `/opt/claude-proxy/.claude/` | ✅ |
| 2026-02-18 | **Filesystem permisos workspace** — `o+rwX` recursivo en workspace + `.git/` ownership a `claude-proxy`. OpenClaw (`ubuntu`) sigue funcionando via permisos world-writable | VPS filesystem | ✅ |
| 2026-02-19 | **Fix permisos claude-proxy memory** — `chown -R claude-proxy:claude-proxy /opt/claude-proxy/.claude/projects/` (era root:root) | VPS | ✅ |
| 2026-02-19 | **Fix crash Maya: sesión E2BIG** — Sesión `d9fbc89d` creció a 352KB (OAuth work), excedió ARG_MAX del kernel → `spawn E2BIG`. Archivada y limpiada | VPS sessions | ✅ |
| 2026-02-19 | **Fix crash Maya: auth conflict** — Maya creó `auth.json` + `auth-profiles.json` de google-antigravity en `/opt/claude-proxy/.openclaw/`, sobreescribiendo OAuth token válido → `code -13` (SIGPIPE). Auth files movidos a backup | VPS claude-proxy | ✅ |
| 2026-02-19 | **Fix permisos Docker restart** — Docker `compose restart` resetea `chmod` de `.openclaw/` a `drwx------`. Añadido `ExecStartPre=+/bin/chmod o+rx` al systemd service para fix automático | VPS systemd | ✅ |
| 2026-02-19 | **Contexto preservado** — Guardado `memory/2026-02-19-session-context.md` con estado del trabajo OAuth de Maya antes de limpiar sesión | VPS memory | ✅ |
| 2026-02-19 | **Fix code -13 definitivo** — `RestrictAddressFamilies` ampliado con `AF_INET6 AF_NETLINK` (Claude CLI necesita IPv6/DNS). Era la causa real del SIGPIPE | VPS systemd | ✅ |
| 2026-02-19 | **Write access gateway agent dir** — `ReadWritePaths` ampliado + `ExecStartPre` chgrp/chmod 770 en `.../agents/main/agent/` para que Maya pueda escribir auth-profiles.json | VPS systemd | ✅ |
| 2026-02-19 | **Fix zombie processes** — `systemctl restart` no mata hijos zombie del proxy → `code -13` persiste. Fix: usar `stop + sleep 2 + start` | VPS systemd | ✅ |
| 2026-02-19 | **Acceso escritura openclaw.json** — Permisos `660 ubuntu:claude-proxy` para que Maya pueda modificar la config del gateway | VPS filesystem | ✅ |
| 2026-02-19 | **Config Antigravity models** — Añadidos `google-antigravity/claude-opus-4-6-thinking`, `claude-sonnet-4-5`, `gemini-3-pro` + auth-profiles.json restaurado de backup | openclaw.json | ✅ |
| 2026-02-19 | **CLAUDE.md creado** — Instrucciones permanentes inyectadas automáticamente en cada sesión de Claude Code CLI. Contiene: arquitectura tools (sandbox vs gateway), reglas filesystem, procedimiento restart. Maya ya tiene "propiocepción" de su cuerpo (OpenClaw) | VPS workspace | ✅ |
| 2026-02-19 | **Diagnóstico definitivo proxy** — El proxy solo pasa texto plano. OpenClaw tools NO son interceptadas automáticamente. Maya no puede invocar tools de OpenClaw "nativamente" via el proxy | VPS proxy | ✅ |
| 2026-02-19 | **`ocl-tool` wrapper creado** — `/usr/local/bin/ocl-tool` llama al gateway interno (127.0.0.1:18789) via `docker exec`. Maya usa `sudo ocl-tool <tool> '<args_json>'` para invocar tools del gateway | VPS | ✅ |
| 2026-02-19 | **Sudoers rules** — `/etc/sudoers.d/claude-proxy-ocl`: claude-proxy puede ejecutar `sudo ocl-tool` y `sudo docker exec openclaw-pyi4-openclaw-1 *` sin contraseña | VPS | ✅ |
| 2026-02-19 | **sessions_spawn habilitado en HTTP API** — `gateway.tools.http.allow: [sessions_spawn, sessions_send]` en openclaw.json (por defecto estaban en deny list del API) | openclaw.json | ✅ |
| 2026-02-19 | **CLAUDE.md reescrito** — Documentación corregida con la arquitectura REAL: 3 tipos de tools (SDK sandbox, ocl-tool gateway, docker exec container) | VPS workspace | ✅ |
| 2026-02-19 | **Fix ocl-tool sin sudo** — `claude-proxy` añadido al grupo `docker` → puede usar `docker exec` directamente. `NoNewPrivileges=true` bloqueaba sudo. CLAUDE.md actualizado (sin sudo). | VPS | ✅ |
| 2026-02-19 | **Worker agent creado** — Multi-agent config: Matrix (`claude-max/claude-sonnet-4`) añadido a `agents.list`, `bindings`, `agentToAgent` habilitado | openclaw.json | ✅ |
| 2026-02-19 | **Matrix workspace** — `workspace-worker/SOUL.md` creado con personalidad de brazo ejecutor | VPS | ✅ |
| 2026-02-19 | **Telegram multi-bot** — Bot `@Matrix_maya_bot` creado y conectado. Estructura `accounts` con default=Maya, worker=Matrix | openclaw.json | ✅ |
| 2026-02-19 | **OpenClaw actualizado v2026.2.17** — Fix bug multi-agent session path traversal (`resolvePathWithinSessionsDir` usaba dir del main para worker → `../..` trigger). También fix `gateway.tools` key inválido, `session.dmScope: per-account-channel-peer` | VPS Docker | ✅ |
| 2026-02-20 | **Grupo Telegram de coordinación creado** — ID `-1003600657980`. Maya + Matrix + Charly en el mismo grupo. `requireMention: true` para el grupo (cada bot responde solo cuando se le menciona) | Telegram / openclaw.json | ✅ |
| 2026-02-20 | **CLAUDE.md Maya: REGLA #7 + #8** — #7: Maya confirma recepción de tareas Anti-Gravity a Charly por Telegram antes de ejecutar. #8: Maya puede delegar a Matrix via `sessions_spawn` + reporta en grupo | VPS workspace/CLAUDE.md | ✅ |
| 2026-02-20 | **SOUL.md Maya + Matrix: sección coordinación grupo** — Instrucciones para notificar en grupo al delegar/completar tareas. accountId correcto por agente (default=Maya, worker=Matrix) | VPS workspace/SOUL.md | ✅ |
| 2026-02-20 | **Fix agentToAgent vacío** — `agents.agentToAgent: {enabled:true, allow:[main,worker]}` estaba vacío. Corregido. | openclaw.json | ✅ |
| 2026-02-20 | **Fix gateway.tools.http.allow vacío** — Habilitados `sessions_spawn, sessions_send, sessions_list, session_status, agents_list` en HTTP API | openclaw.json | ✅ |
| 2026-02-20 | **Comunicación Maya→Matrix real** — `sessions_spawn` via ocl-tool NO funciona para agentToAgent. El método real: `docker exec openclaw-pyi4-openclaw-1 openclaw agent --agent worker --message "tarea" --channel telegram --to "[REDACTED]" --deliver --timeout 600` | VPS | ✅ |
| 2026-02-20 | **IDENTITY.md Matrix rellenado** — Estaba en blanco (template vacío), causaba confusión de identidad. Rellenado con nombre, bot (@Matrix_maya_bot), rol y reglas claras | VPS workspace-worker | ✅ |
| 2026-02-20 | **SOUL.md Matrix: prefijo identidad** — Añadido al inicio: "SOY MATRIX (@Matrix_maya_bot), NO MAYA (@Maya_Matrix_bot)" para evitar confusión en grupo | VPS workspace-worker/SOUL.md | ✅ |
| 2026-02-20 | **Shared Brain (Supabase) — pendiente** — Arquitectura diseñada: Supabase como cerebro compartido de todos los agentes (Anti-Gravity, Maya, Matrix, Claude Code sandbox). Pendiente creación proyecto Supabase (delegado a Matrix cuando esté estable) | — | ⏳ |
| 2026-02-20 | **Clonado recursos Maya→Matrix** — Copiados al workspace-worker: skills (google-oauth, himalaya, n8n-control, sqlite-storage, n8n-workflow-automation, web-search), Google OAuth tokens, `.env.email`, HEARTBEAT.md, memory files | VPS workspace-worker | ✅ |
| 2026-02-20 | **Fix gateway crash — 3 keys inválidos** — Maya introdujo `agents.agentToAgent`, `groups.accountOverrides`, `gateway.tools.http` durante sus sesiones. Gateway salía con code 1. Limpiados manualmente | openclaw.json | ✅ |
| 2026-02-20 | **Group config corregido** — Cambiado de `groupPolicy: allowlist` (sin IDs) a estructura `groups` oficial: `groups."*": {requireMention: true}`, `groups."-1003600657980": {requireMention: false, allowFrom: [[REDACTED]]}` | openclaw.json | ✅ |
| 2026-02-20 | **Session locks recurrentes** — Limpiados `.lock` files en `agents/worker/sessions/` que bloqueaban Matrix con `session file locked (timeout 10000ms)`. Problema recurrente tras crashes | VPS | ✅ |
| 2026-02-20 | **sessions_spawn/sessions_send desbloqueados en HTTP API** — `gateway.tools.allow: ["sessions_spawn","sessions_send"]` añadido. Maya puede delegar tareas a Matrix via `ocl-tool sessions_spawn '{"task":"...","agentId":"worker"}'` | openclaw.json | ✅ |
| 2026-02-20 | **Comunicación bidireccional Maya↔Matrix** — Maya usa `ocl-tool` (HTTP API), Matrix usa `sessions_spawn` nativo. `subagents.allowAgents` configurado mutuamente (`main↔worker`) | openclaw.json + SOUL.md | ✅ |
| 2026-02-20 | **CLAUDE.md Maya: REGLA #9** — Prohibido modificar `openclaw.json` directamente (keys inválidos crashean gateway). Corregido `"agent"` → `"agentId"` en ejemplos de sessions_spawn | VPS workspace/CLAUDE.md | ✅ |
| 2026-02-20 | **SOUL.md Matrix: instrucciones comms** — Añadida sección de comunicación bidireccional con Maya + regla no modificar config | VPS workspace-worker/SOUL.md | ✅ |
| 2026-02-20 | **Limitación Telegram documentada** — Bots NO pueden ver mensajes de otros bots en grupos (limitación Bot API). Maya↔Matrix se comunican via agentToAgent interno, no via grupo | Documentación | ✅ |
| 2026-02-21 | **Fix config inválida Matrix** — Eliminados keys no reconocidos en `agents.list[1].subagents` (`maxConcurrent`, `archiveAfterMinutes`). Causaban que cada config reload fallara | openclaw.json | ✅ |
| 2026-02-21 | **Fix identidad Matrix (CAUSA RAÍZ)** — `models.json` del worker contenía TODOS los modelos de Maya (claude-max, opus, sonnet, haiku, antigravity). Kimi K2.5 veía "Claude Opus 4.6" en su catálogo y se identificaba como Maya. **Reescrito para solo contener Kimi K2.5** | VPS agents/worker/agent/models.json | ✅ |
| 2026-02-21 | **Limpieza workspace-worker** — Eliminados memory files viejos copiados de Maya (2026-02-12, 2026-02-13) y directorio backup/ que reforzaban confusión de identidad. Limpiadas 16 sesiones stale | VPS workspace-worker | ✅ |
| 2026-02-21 | **Workflow `/github-sync` creado** — Proceso completo para sincronizar docs con GitHub: redacción de credenciales, push a repos, troubleshooting. Soporta push a ambos repos, solo Charly103, o solo Maya | `~/.agent/workflows/github-sync.md` | ✅ |
| 2026-02-21 | **GitHub Maya (`deepwbmayamatrix-ai/Maya`) inicializado** — Repo vacío inicializado con ORQUESTADOR.md (redactado) + error-journal.md completo (25 entradas). Token: `/data/.openclaw/workspace/data/.github_token` | GitHub | ✅ |
| 2026-02-21 | **GitHub Charly (`Charly103/maya-agent-army`) actualizado** — ORQUESTADOR.md redactado (503 líneas), error-journal.md (282 líneas), agent identity files (Maya SOUL + Matrix SOUL/IDENTITY/AGENTS/MEMORY). Todo sin credenciales | GitHub | ✅ |
| 2026-02-21 | **Sistema de redacción seguro** — `~/maya/.redact-rules.sed` contiene reglas de sustitución de credenciales. ORQUESTADOR redactado se genera con `sed -f`. Verificación post-redacción con grep. Ningún token/key/phone expuesto en repos | Local | ✅ |

---

## 11. ⚠️ Zonas de Peligro

| Archivo | Riesgo | Precaución |
|---------|--------|------------|
| `openclaw.json` | **CRÍTICO** — Keys no reconocidos crashean el gateway (exit 1) | **Solo Anti-Gravity modifica este archivo.** Maya y Matrix tienen prohibido editarlo (REGLA #9). Siempre verificar con `openclaw doctor` después de cambios |
| `openclaw.json` | Cambios se aplican en caliente | Hacer backup antes de editar. Usar Python, no sed |
| `SOUL.md` | Define personalidad de Maya | No sobreescribir sin leer actual primero |
| `.env` / `.env.email` | API keys sensibles | Nunca en Git ni logs. Chmod 600 |
| `/etc/claude-proxy/env` | OAuth token Max plan | chmod 600, solo `claude-proxy` puede leer. Si se compromete, revocar en claude.ai |
| Skill `gog` | Acceso completo a Gmail, Calendar, Drive | Cuidado con datos personales |
| Tool `nodes` | Control de hardware | NUNCA habilitar |
| `skills/google-oauth/config/tokens.json` | Tokens OAuth | Permanentes (app publicada) |
| `/opt/claude-proxy/.openclaw/` | Sandbox auth de Claude Code | Maya NO debe crear auth files aquí — conflicto con proxy OAuth |
| `docker compose restart` | Resetea permisos de `.openclaw/` | SIEMPRE seguir con `systemctl restart claude-proxy` |
| Browser / Chromium | Crash deja procesos huérfanos + locks | Tras crash: `pkill -f chromium`, borrar `*.lock` y `Singleton*` en `/data/.openclaw/browser/` y `/data/.openclaw/agents/*/sessions/` |
| `claude-proxy.service` | Usa token Max plan de Charly, cwd = workspace Maya | No exponer puerto 3456 al exterior. Solo bridge Docker. El cwd apunta a `/docker/openclaw-pyi4/data/.openclaw/workspace` — no cambiar sin actualizar `ExecStart` (usa path absoluto) |

---

## 12. Roadmap

- [x] Publicar app OAuth en Google Cloud (tokens permanentes)
- [x] Explorar Heartbeat system para tareas proactivas → `every: 60m` ✅
- [x] Agregar MCP servers a Maya (filesystem, memory) via mcporter ✅
- [x] Crear skill n8n-control para Maya ✅
- [ ] Integrar ElevenLabs para TTS (skill `sag`)
- [ ] Crear skill personalizado de YouTube analytics
- [ ] Configurar Daily Brief (cron matutino via Telegram)
- [ ] Configurar grupo Telegram con topics para tareas paralelas
- [ ] Conectar n8n con Maya via MCP (necesita API key)
- [ ] Crear skill orquestador dentro de Maya (VPS)
- [ ] Agregar más MCP servers (GitHub, PostgreSQL, etc.)
- [x] Crear Worker agent (brazo ejecutor) multi-agent ✅
- [x] Completar Matrix: crear bot Telegram en @BotFather y añadir token ✅
- [x] Crear grupo Telegram de coordinación (Maya + Matrix + Charly) ✅
- [x] Fix identidad Matrix (IDENTITY.md + SOUL.md prefix + models.json limpiado) ✅
- [x] Fix agentToAgent + gateway.tools.http.allow ✅
- [ ] Test E2E completo: Maya → Matrix delegación de tarea con reporte en grupo
- [ ] Shared Brain Supabase: crear proyecto + schema + skill para todos los agentes
- [ ] Conectar Anti-Gravity al Shared Brain via hook de startup + Supabase REST

---

## 13. Comunicación Maya ↔ Matrix (verificado 2026-02-20)

### Método real para Maya → Matrix
`sessions_spawn` via ocl-tool HTTP API NO funciona para agentToAgent entre agentes principales.
El método correcto (verificado funcionando):
```bash
docker exec openclaw-pyi4-openclaw-1 openclaw agent \
  --agent worker \
  --message "descripción de la tarea" \
  --channel telegram --to "[REDACTED]" \
  --deliver --timeout 600
```

### Grupo Telegram de coordinación
- **ID**: `-1003600657980`
- **Participantes**: Charly + @Maya_Matrix_bot + @Matrix_maya_bot
- **Config**: `requireMention: true` (cada bot solo responde cuando se le menciona)
- **Maya escribe en grupo**: `ocl-tool message '{"channel":"telegram","accountId":"default","to":"-1003600657980","text":"..."}'`
- **Matrix escribe en grupo**: `ocl-tool message '{"channel":"telegram","accountId":"worker","to":"-1003600657980","text":"..."}'`

### Flujo de delegación
```
Charly/Anti-Gravity → Maya (DM o grupo) → Maya delega a Matrix (openclaw agent)
Matrix ejecuta → reporta en grupo → Maya resume a Charly
```

---

## 14. Otros Servicios en VPS

| Servicio | Puerto | Estado |
|----------|--------|--------|
| n8n | `127.0.0.1:5678` | ✅ Activo |
| Traefik | 80/443 | ✅ Activo |
| PostgreSQL | 5432 | 🔒 Interno |

---

---

## 15. GitHub — Repositorios del Ecosistema

| Repo | Owner | Contenido | Privado |
|------|-------|-----------|--------|
| [`maya-agent-army`](https://github.com/Charly103/maya-agent-army) | `Charly103` | Docs completos + agent identity files | Sí |
| [`Maya`](https://github.com/deepwbmayamatrix-ai/Maya) | `deepwbmayamatrix-ai` | ORQUESTADOR + error-journal | No |

### Cuentas GitHub
| Cuenta | Rol | Token |
|--------|-----|-------|
| `Charly103` | Personal de Charly | Pedir a Charly si se necesita |
| `deepwbmayamatrix-ai` | Ecosystem (Maya/Matrix) | VPS: `/data/.openclaw/workspace/data/.github_token` |

### Sincronización
- Workflow: `/github-sync` en `~/.agent/workflows/github-sync.md`
- Redacción: `~/maya/.redact-rules.sed` (credenciales → `[REDACTED]`)
- **NUNCA** subir ORQUESTADOR.md sin redactar

---

*Este documento debe ser actualizado al final de cada sesión de Anti-Gravity.*
*Usa `/orquestador` para iniciar una sesión con contexto completo.*
