# 🧠 Antigravity — Error Journal

> Este archivo es mi memoria de errores. Lo consulto ANTES de actuar en tareas complejas.
> Cada entrada es una lección real que me ahorrará tiempo y errores en el futuro.
> Formato: categoría, qué pasó, por qué, qué hacer diferente.

---

## Categorías de Errores
- **[PROACTIVIDAD]** — Cosas que debí hacer sin que me lo pidieran
- **[SSH/DOCKER]** — Errores de escaping y ejecución remota
- **[INVESTIGACIÓN]** — Errores de enfoque al investigar tecnologías
- **[CONFIG]** — Errores de configuración de herramientas
- **[COMUNICACIÓN]** — Errores al comunicarme con el usuario
- **[ARQUITECTURA]** — Errores de diseño o entendimiento de sistemas

---

## Entradas

### 001 — [PROACTIVIDAD] No consulté la documentación oficial por mi cuenta
- **Fecha**: 2026-02-14
- **Contexto**: Tarea de restaurar MCP en OpenClaw. Intenté múltiples enfoques a ciegas antes de que el usuario me dijera que revisara la documentación oficial.
- **Error**: Esperé a que el usuario me sugiriera consultar docs.openclaw.ai en lugar de hacerlo proactivamente.
- **Lección**: **SIEMPRE** que trabaje con un sistema, herramienta o tecnología — lo PRIMERO es ir a la documentación oficial. No esperar a que el usuario me lo diga. Esto es básico y debo deducirlo yo.
- **Regla**: Ante cualquier sistema desconocido o error inesperado → docs oficiales PRIMERO, experimentar DESPUÉS.
- **Impacto**: Perdí ~30 minutos probando cosas a ciegas que la documentación habría resuelto en 5.

### 002 — [SSH/DOCKER] Escaping de JSON a través de SSH+Docker+Bash
- **Fecha**: 2026-02-14
- **Contexto**: Intentar pasar argumentos JSON a `mcporter call` a través de capas ssh→docker exec→bash.
- **Error**: Las comillas JSON se destruían al pasar por 3+ capas de interpretación de shell.
- **Lección**: Usar la sintaxis nativa de la herramienta (`key=value`) en lugar de JSON cuando se pasa por múltiples capas de shell. Alternativamente, escribir el JSON a un fichero temporal y luego usarlo.
- **Regla**: Evitar JSON inline en comandos SSH+Docker. Preferir: (1) `key=value`, (2) heredoc a fichero, (3) base64 encode/decode.

### 003 — [SSH/DOCKER] Heredoc a través de SSH con caracteres especiales
- **Fecha**: 2026-02-14
- **Contexto**: Crear un archivo skill en el container con heredoc a través de SSH.
- **Error**: El heredoc con backticks, `<`, `>` y otros caracteres especiales de markdown se interpretaba localmente en lugar de remotamente.
- **Lección**: Para crear archivos en containers remotos con contenido complejo: (1) crear el archivo localmente, (2) `docker cp` al container, o (3) usar `base64` encode/decode. Los heredoc a través de SSH son frágiles.
- **Regla**: Contenido complejo → `docker cp` o `base64`. Contenido simple → heredoc está bien.

### 004 — [INVESTIGACIÓN] Asumir que un concepto funciona igual en todos los sistemas
- **Fecha**: 2026-02-14
- **Contexto**: Asumí que `mcpServers` en el código de OpenClaw significaba que OpenClaw podía consumir MCP servers directamente.
- **Error**: `mcpServers` en el código ACP era sobre recibir configuración de editores externos (Cursor/Zed), NO sobre consumir MCP servers. El código explícitamente decía "ignoring N MCP servers".
- **Lección**: Leer el CÓDIGO antes de asumir funcionalidad. Un nombre de variable no implica que la feature funciona como espero. Buscar la implementación real.
- **Regla**: Variable/propiedad con nombre conocido ≠ feature implementada. Verificar siempre el flujo completo.

### 005 — [CONFIG] El paquete npm @modelcontextprotocol/server-fetch no existe
- **Fecha**: 2026-02-14
- **Contexto**: Asumí que existía un paquete npm para el MCP fetch server igual que para filesystem y memory.
- **Error**: El MCP fetch server (`mcp-server-fetch`) es un paquete Python/PyPI, no npm.
- **Lección**: Verificar que un paquete existe ANTES de configurarlo. No asumir convenciones de naming entre ecosistemas.
- **Regla**: Antes de `npm install X` o `npx X` → verificar en npm que el paquete existe.

### 006 — [ARQUITECTURA] Confundir archivos del HOST con archivos del CONTAINER
- **Fecha**: 2026-02-14
- **Contexto**: Configuré la API key de n8n en `/data/.openclaw/workspace/.env.email` DENTRO del container, y el skill decía "lee de variable de entorno". Pero Docker Compose carga env vars desde `/docker/openclaw-pyi4/.env.email` en el HOST.
- **Error**: No entendí que hay DOS archivos `.env.email` distintos: uno en el HOST (que Docker Compose lee como `env_file:`) y otro DENTRO del container (que es solo un archivo de texto). Poner la key en el del container no la convierte en variable de entorno.
- **Lección**: Las variables de entorno en Docker vienen de: (1) `env_file:` en docker-compose.yml (archivos del HOST), (2) `environment:` en docker-compose.yml, (3) `docker run -e`. Los archivos dentro del container son solo archivos, NO se cargan como env vars.
- **Regla**: Para inyectar env vars en un container → siempre editar el archivo del HOST que está en `env_file:` del docker-compose.yml, luego `docker compose up -d --force-recreate`. Un `restart` NO recarga env_file.
- **Impacto**: Maya no podía acceder a n8n. Múltiples intentos fallidos. Usuario frustrado.

### 007 — [CONFIG] Docker restart NO recarga env_file, solo force-recreate
- **Fecha**: 2026-02-14
- **Contexto**: Tras añadir `N8N_API_KEY` al `.env.email` del host, hice `docker compose restart`.
- **Error**: `docker compose restart` reinicia el proceso dentro del container existente, pero NO relee `env_file:`. Las nuevas variables no se cargaron.
- **Lección**: `restart` = reinicia proceso, `up -d --force-recreate` = recrea container con nuevo env. Para cambios en env_file SIEMPRE usar `docker compose up -d --force-recreate`.
- **Regla**: Cambio en `.env` / `.env.email` / `env_file:` → `docker compose up -d --force-recreate`, NUNCA `restart`.

### 008 — [ARQUITECTURA] Skill de Maya referenciaba credencial sin verificar que fuera accesible
- **Fecha**: 2026-02-14
- **Contexto**: El skill n8n decía "lee de variable de entorno $N8N_API_KEY" pero esa variable no existía en el entorno del container.
- **Error**: Escribí un skill que asumía que una credencial estaría disponible sin verificar que realmente lo estaba. No probé el camino completo: skill → exec → env var → resultado.
- **Lección**: Después de desplegar un skill que referencia credenciales, SIEMPRE verificar end-to-end: (1) la credencial existe donde digo, (2) Maya puede accederla, (3) un comando de ejemplo funciona.
- **Regla**: Todo skill que use credenciales → verificar E2E con `docker exec container env | grep VARIABLE` antes de dar por completado.

### 009 — [INVESTIGACIÓN] Implementar búsqueda web como skill de curl en vez de usar herramienta nativa
- **Fecha**: 2026-02-15
- **Contexto**: El usuario pidió configurar búsqueda web para Maya con Perplexity. Creé un "skill" con instrucciones de curl para llamar a la API de Perplexity.
- **Error**: OpenClaw tiene una herramienta NATIVA `web_search` que se configura en `openclaw.json` bajo `tools.web.search`. Maya la ignoró completamente porque buscaba su herramienta nativa, no instrucciones de curl. Resultado: Maya pidió una API de Brave en vez de usar Perplexity.
- **Lección**: Antes de implementar una feature como "skill manual" (instrucciones con curl, scripts, etc.), verificar si la plataforma ya tiene una herramienta nativa para eso. **LEER LA DOCUMENTACIÓN** de OpenClaw para ver qué herramientas nativas soporta.
- **Regla**: Herramienta nativa > Skill manual. SIEMPRE consultar docs de la plataforma para ver si ya existe soporte nativo antes de improvisar con workarounds.
- **Impacto**: Maya no podía buscar en la web. El usuario tuvo que diagnosticar y corregir el error. Es la SEGUNDA VEZ que no consulto la documentación primero (ver entrada 001).

### 010 — [ARQUITECTURA] Instalar software via brew dentro de Docker rompe dependencias del sistema
- **Fecha**: 2026-02-15
- **Contexto**: El usuario pedía habilitar el skill Gemini en OpenClaw. Hice `brew install gemini-cli` dentro del contenedor Docker. Brew instaló `node 25.6.1` como dependencia, que reemplazó el node del sistema en el PATH (`/data/linuxbrew/.linuxbrew/bin` está antes de `/usr/local/bin`).
- **Error**: El node de brew tiene su propia cadena de certificados SSL que no incluye los del sistema operativo. Resultado: `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` en todas las conexiones HTTPS (Telegram, WhatsApp). OpenClaw entró en crash loop.
- **Lección**: NUNCA instalar paquetes via brew dentro del contenedor sin entender el impacto en las dependencias del sistema (especialmente node, python). Brew puede sobreescribir binarios críticos del sistema.
- **Regla**: Antes de `brew install X` en un contenedor: (1) verificar qué dependencias instalará (`brew deps X`), (2) verificar si alguna es un binario crítico ya presente, (3) consultar la documentación oficial de la plataforma para el método correcto de instalación.
- **Impacto**: OpenClaw en crash loop ~10 minutos. WhatsApp y Telegram desconectados. Tuve que desinstalar todo y force-recrear el contenedor.

### 011 — [PROACTIVIDAD] No consulté la documentación oficial — TERCERA VEZ
- **Fecha**: 2026-02-15
- **Contexto**: En lugar de consultar docs.openclaw.ai para entender cómo habilitar el skill Gemini correctamente, hice `brew install` a ciegas.
- **Error**: Actué sin consultar docs. Es la TERCERA vez que cometo este error (ver #001 y #009).
- **Lección**: CRÍTICO. Este patrón es el error más recurrente. Debo forzarme a consultar documentación ANTES de cualquier acción.
- **Regla**: **OBLIGATORIO**: ante cualquier acción en OpenClaw/VPS → PRIMERO leer docs.openclaw.ai sobre el tema. SIN EXCEPCIONES.

### 012 — [COMUNICACIÓN] Ignorar datos que el usuario ya proporcionó y sobrecomplicar la tarea
- **Fecha**: 2026-02-16
- **Contexto**: El usuario me dio TODA la información nueva (email, password, JSON OAuth, APIs habilitadas) y me pidió simplemente sustituir lo viejo por lo nuevo en todos los sitios.
- **Error**: En vez de hacer la sustitución directa, intenté lanzar un OAuth flow en el browser para "generar tokens", ignorando que el usuario ya me había dado el `client_secret` JSON. Cuando el usuario me dijo que ya me lo había dado, seguí insistiendo. Le hice perder tiempo.
- **Lección**: Cuando el usuario te da datos y te dice "sustituye", SUSTITUYE. No inventes pasos adicionales. Lee lo que te dan, aplícalo, y punto.
- **Regla**: (1) Leer TODOS los datos proporcionados por el usuario ANTES de planificar, (2) Si el usuario dice "ya te lo di" → PARAR y revisar lo que te dieron, (3) No añadir pasos que el usuario no pidió.
- **Impacto**: Frustración del usuario. Pérdida de ~10 minutos con un OAuth flow innecesario.

### 013 — [COMUNICACIÓN] No incluir todas las APIs que el usuario listó
- **Fecha**: 2026-02-16
- **Contexto**: El usuario listó 10 APIs habilitadas (Gmail, Drive, Calendar, Sheets, YouTube x3, Cloud Vision, Dialogflow, Workspace Marketplace).
- **Error**: En el primer pase, solo puse 8 scopes en `.env.email`, olvidando YouTube Reporting y Workspace Marketplace.
- **Lección**: Cuando el usuario te da una lista, cópiala ENTERA. No filtres ni "resumas".
- **Regla**: Contar los items que el usuario da y verificar que el resultado tiene el mismo número. Sin excepciones.

### 014 — [ARQUITECTURA] OAuth de Maya sobreescribió auth de Claude Code proxy → code -13 SIGPIPE
- **Fecha**: 2026-02-19
- **Contexto**: Maya estaba configurando OAuth para modelos google-antigravity dentro de OpenClaw. Para ello, ejecutó código que creó archivos `auth.json` y `auth-profiles.json` en `/opt/claude-proxy/.openclaw/agents/main/agent/`.
- **Error**: Esos archivos de auth de google-antigravity tomaron precedencia sobre el `CLAUDE_CODE_OAUTH_TOKEN` env var que el proxy usaba para autenticarse con Claude Max. Resultado: Claude CLI reportaba `loggedIn: false` → subprocess moría con code -13 (SIGPIPE) → Maya no podía responder a nada.
- **Cadena causal**: OAuth work en sandbox → crea auth files locales → CLI prioriza auth files sobre env var → `Not logged in` → subprocess muere → `code -13` → Maya inoperativa.
- **Lección**: El sandbox de Claude Code (directorio `.claude/` y `.openclaw/` del user claude-proxy) es compartido entre el proxy y las sesiones de Maya. Si Maya modifica archivos de autenticación, puede romper la autenticación del propio proxy.
- **Fix aplicado**: Mover auth files conflictivos a backup (`/opt/claude-proxy/.openclaw-auth-backup/`).
- **Regla**: **NUNCA** dejar que Maya haga operaciones OAuth o de autenticación desde dentro del sandbox de Claude Code. Las configuraciones de auth de terceros deben gestionarse por Antigravity vía SSH, en paths separados, nunca en el home del user claude-proxy.
- **Impacto**: Maya inoperativa ~30 minutos. Múltiples restarts sin efecto hasta diagnosticar la causa raíz.

### 015 — [ARQUITECTURA] Docker `compose restart` resetea permisos de volumes montados
- **Fecha**: 2026-02-19
- **Contexto**: El proxy de Claude Code necesita acceso `o+rx` a `/docker/openclaw-pyi4/data/.openclaw/` y su subdirectorio `workspace/` (propiedad de `ubuntu:ubuntu`). El user `claude-proxy` accede via permisos "others".
- **Error**: Cada `docker compose restart openclaw` resetea los permisos del directorio `.openclaw` a `drwx------` (solo ubuntu), dejando a claude-proxy sin acceso. El proxy falla con `status=200/CHDIR` porque systemd no puede hacer `cd` al `WorkingDirectory`.
- **Lección**: Los permisos `chmod` sobre directorios de volumes Docker NO persisten entre restarts del container. Docker los resetea a los permisos del proceso interno.
- **Fix aplicado**: Añadido `ExecStartPre=+/bin/chmod o+rx ...` al servicio systemd `claude-proxy.service` (el `+` ejecuta como root). Esto fija permisos automáticamente ANTES de que el proxy arranque.
- **Regla**: (1) Cuando un servicio externo necesita acceso a dirs de un volume Docker, usar `ExecStartPre` para fijar permisos. (2) Tras un `docker compose restart`, SIEMPRE reiniciar también `claude-proxy` para que el ExecStartPre vuelva a fijar permisos. (3) Idealmente, crear un oneshot service o path unit que detecte el restart de Docker y reaplique permisos automáticamente.
- **Procedimiento correcto de restart**: `docker compose restart openclaw && systemctl restart claude-proxy` (en ese orden).

### 016 — [CONFIG] `RestrictAddressFamilies` sin IPv6/NETLINK mata subprocess con code -13
- **Fecha**: 2026-02-19
- **Contexto**: Tras fixes de auth y permisos, claude-proxy seguía dando `code -13` (SIGPIPE) con `PID: undefined`.
- **Error**: El systemd service tenía `RestrictAddressFamilies=AF_INET AF_UNIX`, bloqueando `AF_INET6` y `AF_NETLINK`. Claude CLI necesita ambos para resolver DNS y conectar a la API de Anthropic.
- **Fix**: Cambiar a `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX AF_NETLINK`.
- **Regla**: Al hacer hardening de systemd services que hacen llamadas de red, SIEMPRE incluir `AF_INET6` y `AF_NETLINK`.
- **Impacto**: Maya inoperativa ~1h adicional tras múltiples restarts correctos.

### 017 — [CONFIG] `systemctl restart` no mata procesos zombie → code -13 persiste
- **Fecha**: 2026-02-19
- **Contexto**: Tras arreglar auth, permisos y RestrictAddressFamilies, el proxy seguía dando `code -13`. Curl directo también fallaba.
- **Error**: `systemctl restart` reutiliza el cgroup y puede dejar procesos hijos zombie del proxy anterior. El nuevo proxy hereda el estado corrupto y cada subprocess muere inmediatamente con `PID: undefined`.
- **Fix**: Usar `systemctl stop` + `sleep 2` + `systemctl start` (en vez de `restart`). El stop mata TODO el cgroup, el sleep deja que se limpie, y el start arranca limpio.
- **Regla**: Para claude-proxy, SIEMPRE usar `stop + start`, NUNCA `restart`. Documentado en el procedimiento oficial de restart.
- **Impacto**: Maya inoperativa ~30 min adicionales tras "fixes correctos" que no surtían efecto.

### 018 — [ARQUITECTURA] Claude Code CLI no lee MEMORY.md automáticamente — necesita CLAUDE.md
- **Fecha**: 2026-02-19
- **Contexto**: Maya tenía toda la documentación de su arquitectura en MEMORY.md pero seguía confundiéndose sobre sus capacidades en cada sesión nueva.
- **Error**: El proxy usa `claude --print` (modo no-interactivo). En este modo, Claude Code CLI NO lee MEMORY.md del workspace. Sí lee `CLAUDE.md` automáticamente desde su `cwd`.
- **Fix**: Creado `/docker/openclaw-pyi4/data/.openclaw/workspace/CLAUDE.md` con las reglas críticas de arquitectura (tipos de tools, qué ejecuta el sandbox vs el gateway, reglas de filesystem).
- **Regla**: La información que Maya debe tener en CADA sesión va en `CLAUDE.md` (se inyecta automáticamente), no solo en MEMORY.md (que necesita ser leído explícitamente).
- **Impacto**: Maya entraba en bucles de confusión intentando llamar tools de OpenClaw desde Bash.

### 019 — [ARQUITECTURA] CLAUDE.md decía que el gateway intercepta tool calls — FALSO
- **Fecha**: 2026-02-19
- **Contexto**: Creamos CLAUDE.md diciendo "El GATEWAY intercepta y ejecuta las tool calls POR TI". Maya pasó horas intentando invocar tools de OpenClaw directamente en sus respuestas, sin resultado.
- **Error**: El proxy (`openai-to-cli.js`) convierte TODO a texto plano y lo manda a `claude --print`. No hay intercepción de tool calls. El campo `tools` de la request OpenAI es ignorado completamente por el proxy. Claude Code CLI en `--print` mode solo tiene sus propios tools nativos (Bash, Read, Write, etc.).
- **Diagnóstico**: El gateway de OpenClaw escucha en `127.0.0.1:18789` DENTRO del container (loopback, no accesible desde host). El endpoint `/tools/invoke` en ese puerto funciona correctamente via HTTP POST.
- **Fix**: Creado wrapper `/usr/local/bin/ocl-tool` que usa `docker exec openclaw-pyi4-openclaw-1 curl http://127.0.0.1:18789/tools/invoke`. Sudoers rules para claude-proxy. CLAUDE.md reescrito con la arquitectura correcta: 3 tipos de tools (SDK, ocl-tool, docker exec).
- **Regla**: Cuando Maya corre via claude-proxy, NO tiene tools de OpenClaw nativas. Para invocar tools del gateway: `sudo ocl-tool <tool> '<args_json>'`. Para container shell: `sudo docker exec openclaw-pyi4-openclaw-1 <cmd>`. El proxy ES SOLO un puente de texto.
- **Impacto**: ~2 días de confusión y loops porque el CLAUDE.md tenía información incorrecta sobre la arquitectura.
- **Lección**: Antes de documentar "cómo funciona algo", LEER EL CÓDIGO FUENTE del proxy y verificar con pruebas reales. Nunca documentar arquitectura asumida.

### 020 — [CONFIG] claude-proxy `code -13` — Procedimiento rápido de reparación
- **Fecha**: 2026-02-20
- **Contexto**: Maya deja de responder con `Process exited with code -13`. Ocurre periódicamente.
- **Causa**: El proxy pierde la conexión (SIGPIPE) por sesión larga, zombie process, o reset del container.
- **Fix rápido**: 
  ```bash
  ssh root@46.202.171.91 "systemctl stop claude-proxy && sleep 2 && systemctl start claude-proxy"
  ```
  Opcionalmente reiniciar también el gateway si Maya no responde tras el fix del proxy:
  ```bash
  ssh root@46.202.171.91 "cd /docker/openclaw-pyi4 && docker compose restart openclaw"
  ```
- **Regla**: NUNCA usar `systemctl restart` — usar `stop + sleep 2 + start`. Ver entrada #017.
- **Checklist diagnóstico si persiste**:
  1. `systemctl status claude-proxy` — ¿active?
  2. `docker logs --tail 10 openclaw-pyi4-openclaw-1` — ¿ambos bots Telegram activos?
  3. Verificar permisos: `ls -la /docker/openclaw-pyi4/data/.openclaw/` — ¿tiene `o+rx`?
  4. Verificar auth files: `ls /opt/claude-proxy/.openclaw/agents/main/agent/` — ¿hay `auth.json` suelto? Si sí, mover a backup (ver #014)

### 021 — [ARQUITECTURA] OpenClaw multi-agent session path traversal bug (v2026.2.12)
- **Fecha**: 2026-02-19

### 022 — [ARQUITECTURA] Maya modifica openclaw.json con keys inválidos → gateway crash
- **Fecha**: 2026-02-20
- **Contexto**: Maya intentaba habilitar `sessions_spawn` via HTTP API. Editó `openclaw.json` añadiendo `gateway.tools.allow`, `agents.agentToAgent`, y `groups.accountOverrides` — todos inválidos o mal ubicados.
- **Error**: OpenClaw valida el config al arranque y crashea con exit 1 si encuentra keys no reconocidos. Maya lo hizo 3 veces en la misma sesión, cada vez añadiendo keys diferentes.
- **Fix**: Limpiados manualmente. Añadida **REGLA #9 en CLAUDE.md**: Maya tiene PROHIBIDO editar openclaw.json. Solo Anti-Gravity puede modificar la config.
- **Config correcta que SÍ funcionó**: `gateway.tools.allow: ["sessions_spawn", "sessions_send"]` (key oficialmente documentado en docs.openclaw.ai/gateway/tools-invoke-http-api)
- **Regla**: Solo Anti-Gravity modifica `openclaw.json`. Maya y Matrix no deben tocarlo nunca.

### 023 — [SSH/DOCKER] JSON escaping con ocl-tool sessions_spawn
- **Fecha**: 2026-02-20
- **Contexto**: `ocl-tool sessions_spawn '{"task":"...","agent":"worker"}'` devolvía `task required`.
- **Error**: El parámetro correcto es `agentId`, no `agent`. Además, el JSON pasaba por bash→python→docker exec→curl, perdiendo los args.
- **Fix**: Corregido `"agent"` → `"agentId"` en CLAUDE.md. El `ocl-tool` wrapper usa python3 para construir el payload, funciona correctamente cuando se llama localmente en el VPS (no a través de SSH).
- **Regla**: El param es `agentId` (no `agent`). Para probar desde SSH, usar `docker exec ... curl` directamente en vez de `ocl-tool`.
- **Contexto**: Configurando segundo agente (Matrix) en OpenClaw multi-agent routing.
- **Error**: `Session file path must be within sessions directory` — OpenClaw v2026.2.12 tenía un bug donde `resolvePathWithinSessionsDir()` usaba el directorio de sesiones del agente `main` como base para resolver paths de sesiones del agente `worker`. El path relativo resultante era `../../worker/sessions/...` → triggereaba la protección anti-traversal.
- **Fix**: Actualizar OpenClaw. `docker compose pull && docker compose up -d` → v2026.2.17+ arregla esto.
- **Regla**: Ante errores de "session file path" en multi-agent, primero verificar versión de OpenClaw y actualizar.
- **Config correcta multi-agent Telegram**:
  - NO poner `botToken`/`dmPolicy`/`allowFrom` a nivel raíz de `channels.telegram` — solo dentro de `accounts`
  - Añadir `session.dmScope: "per-account-channel-peer"` para multi-account
  - Cada agente necesita directorio `agents/<id>/agent/` y `agents/<id>/sessions/`

---

## Patrones Recurrentes (actualizar cuando se repita un patrón)

| Patrón | Frecuencia | Acción preventiva |
|--------|-----------|-------------------|
| No consultar docs oficiales primero | **3 veces** | SIEMPRE empezar por docs — TRIPLE REINCIDENTE |
| Problemas de escaping en SSH+Docker | 2 veces | Usar key=value o docker cp |
| Conflictos auth en sandbox compartido | 1 vez | OAuth/auth de terceros NUNCA desde el sandbox de claude-proxy |
| Docker restart resetea permisos | 1 vez | ExecStartPre + reiniciar proxy después de Docker |
| Hardening systemd bloquea red IPv6/DNS | 1 vez | Siempre incluir AF_INET6 + AF_NETLINK en RestrictAddressFamilies |
| systemctl restart deja zombies | 1 vez | SIEMPRE stop + start, NUNCA restart para claude-proxy |
| CLI no lee MEMORY.md automáticamente | 1 vez | Info crítica va en CLAUDE.md (se auto-inyecta), no solo en MEMORY.md |
| Documentar arquitectura sin verificarla | 1 vez | LEER el código fuente del proxy ANTES de escribir CLAUDE.md. Verificar con pruebas reales |
| Proxy ignora campo tools — solo texto | 1 vez | Para OpenClaw tools: `sudo ocl-tool`. Para container: `sudo docker exec`. El proxy solo pasa texto |
| Sobrecomplicar tareas simples | 1 vez | Si el usuario dice "sustituye X por Y" → HAZLO. No inventes más pasos |
| Ignorar/perder datos del usuario | 1 vez | Leer TODO lo que el usuario da ANTES de actuar |
| Asumir features sin leer código | 1 vez | Leer implementación real |
| Workaround manual vs herramienta nativa | 1 vez | Verificar si la plataforma tiene soporte nativo ANTES de improvisar |
| Asumir existencia de paquetes | 1 vez | Verificar antes de configurar |
| Confundir HOST vs CONTAINER files | 1 vez | env_file = HOST, volume = CONTAINER |
| Instalar software que rompe deps del sistema | 1 vez | NUNCA brew install en containers sin verificar deps |
| No verificar E2E tras desplegar | 1 vez | Siempre probar el camino completo |
| docker restart vs force-recreate | 1 vez | env changes → force-recreate |
| claude-proxy code -13 periódico | **recurrente** | `stop + sleep 2 + start` (NUNCA restart). Ver #017, #020 |
| Multi-agent session path error | 1 vez | Actualizar OpenClaw + verificar config multi-account. Ver #021 |
| Maya edita openclaw.json → gateway crash | **3 veces** | REGLA #9 en CLAUDE.md: Maya y Matrix **NUNCA** editan config. Solo Anti-Gravity. Ver #022 |
| Session locks tras crash | **recurrente** | `find .../agents/ -name "*.lock" -delete` + restart gateway |
| models.json compartido contamina identidad | 1 vez | Cada agente debe tener su PROPIO `models.json` con SOLO sus modelos. Ver #024 |
| Config con keys inválidos → reload falla silenciosamente | 1 vez | Verificar logs post-cambio. Ver #025 |

---

## Meta-Reglas (las más importantes)

1. **Documentación primero, experimentar después** — Nunca actuar a ciegas cuando existe documentación oficial. **TRIPLE REINCIDENTE: ya fallé en esto 3 veces.**
2. **Verificar antes de asumir** — Que algo se llame `mcpServers` no significa que haga lo que creo.
3. **Simplicidad en la ejecución remota** — Cuantas menos capas de interpretación, menos errores.
4. **Ser proactivo, no reactivo** — El usuario no debería tener que decirme cosas obvias como "revisa la documentación".
5. **HOST ≠ CONTAINER** — env_file de Docker Compose lee del HOST. Archivos dentro del container son solo archivos. restart ≠ recreate.
6. **Verificación E2E obligatoria** — Después de desplegar cualquier config/skill con credenciales, verificar que funciona end-to-end antes de decir "listo".
7. **Nativo > Manual** — Antes de crear un skill/workaround, verificar si la plataforma ya tiene una herramienta nativa. Los skills de curl/scripts son el último recurso, no el primero.
8. **Escuchar al usuario** — Si el usuario te da datos, ÚSALOS. Si dice "ya te lo di", PARA y revisa. No inventes pasos extra. Sustituir = sustituir, no "mejorar".
9. **Sandbox compartido = zona de peligro** — El home de claude-proxy es compartido. Maya NO debe crear/modificar auth files ahí. Configuraciones de terceros van por SSH desde Antigravity.
10. **Docker restart = permisos volátiles** — Tras restart de Docker, los permisos `chmod` se pierden. Usar systemd `ExecStartPre` para refix automático. Procedimiento: `docker compose restart` → `systemctl restart claude-proxy`.
11. **Leer código antes de documentar arquitectura** — Si escribes instrucciones sobre "cómo funciona algo" (CLAUDE.md, SOUL.md), léete el código fuente PRIMERO. La arquitectura asumida puede ser completamente errónea y causar días de bucles.
12. **Proxy claude-max = SOLO texto** — El proxy convierte todo a texto plano. Las tools de OpenClaw NO se interceptan. Para tools del gateway: `sudo ocl-tool`. Para el container: `sudo docker exec openclaw-pyi4-openclaw-1`.
13. **code -13 = stop + sleep 2 + start** — Cuando Maya muere con `Process exited with code -13`, ejecutar: `ssh root@46.202.171.91 "systemctl stop claude-proxy && sleep 2 && systemctl start claude-proxy"`. Si persiste, reiniciar también el gateway.
14. **models.json por agente = AISLADO** — Cada agente en multi-agent DEBE tener su propio `models.json` con SOLO los modelos que le corresponden. Si el worker ve modelos de Maya (claude-max/claude-opus-4-6), el LLM se confundirá y creerá que ES Maya.

---

### 024 — [ARQUITECTURA] models.json compartido causa confusión de identidad en Matrix
- **Fecha**: 2026-02-21
- **Contexto**: Matrix (Kimi K2.5) respondía diciendo que era Maya y usaba Claude Opus 4.6. Los archivos de identidad (SOUL.md, IDENTITY.md, AGENTS.md, MEMORY.md) estaban correctos y explícitamente decían "Soy Matrix".
- **Error**: El `models.json` en `agents/worker/agent/` contenía TODOS los modelos del ecosistema: `claude-max/claude-opus-4-6`, `claude-sonnet-4`, `claude-haiku-4`, `google-antigravity/claude-opus-4-6-thinking`, `gemini-3-pro`, etc. Kimi K2.5 veía "Claude Opus 4.6 (Max Plan - Most Powerful)" en su catálogo de modelos disponibles y se identificaba como ese modelo.
- **Causa**: Al clonar recursos de Maya a Matrix (2026-02-20), se copió el `models.json` completo incluyendo providers que no le corresponden.
- **Fix**: Reescrito `models.json` del worker para contener SOLO el provider `openrouter` con el modelo `moonshotai/kimi-k2.5`.
- **Regla**: En multi-agent, cada agente debe tener su PROPIO `models.json` con SOLO los modelos que usa. Los modelos de otros agentes NO deben aparecer en su catálogo, porque el LLM los interpreta como propios.
- **Impacto**: Matrix inutilizable durante >24h. Múltiples intentos de fix (IDENTITY.md, SOUL.md prefixes) no funcionaban porque la causa raíz era el catálogo de modelos, no los archivos de identidad.

### 025 — [CONFIG] Keys inválidos en worker subagents causan reload silencioso
- **Fecha**: 2026-02-21
- **Contexto**: `openclaw.json` tenía `maxConcurrent` y `archiveAfterMinutes` en `agents.list[1].subagents` (worker).
- **Error**: Estos keys son válidos en `agents.defaults.subagents` pero NO en `agents.list[].subagents` individual. OpenClaw rechazaba el config reload silenciosamente con `Unrecognized keys` en logs.
- **Impacto**: Todos los cambios de config posteriores (incluyendo fixes de identidad) NO se aplicaban porque el reload fallaba.
- **Regla**: Después de modificar `openclaw.json`, SIEMPRE verificar logs con `docker logs --tail 5 openclaw-pyi4-openclaw-1` para confirmar que el reload fue aceptado. El error `Unrecognized keys` es silencioso — no crashea, simplemente ignora el cambio.
