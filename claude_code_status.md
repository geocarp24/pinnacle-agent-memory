# CLAUDE CODE STATUS — Sistema ALEX
> Última actualización: 2026-04-05 | Servidor VPS: 187.77.215.146

---

## 1. CONFIGURACIÓN DEL SERVIDOR VPS

| Campo | Valor |
|-------|-------|
| **IP** | 187.77.215.146 |
| **OS** | Linux Ubuntu |
| **Claude Code CLI** | v2.1.92 (en `/usr/local/bin/claude`) |
| **Modelo activo** | claude-sonnet-4-6 |
| **Shell** | bash |
| **Working directory** | `/opt/alex-bot` |

---

## 2. RUTAS IMPORTANTES DEL SISTEMA

```
/opt/alex-bot/                    → Repo principal ALEX (geocarp24/alex-real-estate-system)
├── agents/                       → Prompts y memorias de sub-agentes
│   ├── scout.md / scout.agent.md
│   ├── matematico.md / matematico.agent.md
│   ├── fact-checker.md / fact-checker.agent.md
│   ├── tracy.md
│   ├── social_media.md
│   ├── airtable.md
│   ├── protocolo_seguro.md
│   ├── cola_mensajes.md          → Canal inter-agente
│   ├── memoria_*.md              → Memorias de cada sub-agente
│   └── alerta_telegram.sh        → Script de alertas
├── telegram_bot/
│   ├── alex_bot.py               → Bot principal de Telegram
│   ├── telegram_memory.md        → Memoria de sesiones Telegram
│   └── requirements.txt
├── hostinger/                    → PHP deployado en agents.pinnaclegroupwi.com
│   ├── agents/
│   │   ├── github_bridge.php     → Lee archivos de GitHub (GET)
│   │   └── github_write.php      → Escribe archivos en GitHub (POST/PUT)
│   └── tools/
│       ├── el_polling.php        → Cron Tracy skip trace (cada 5min)
│       └── el_chismoso.php       → Webhook Tracy → Contacts (Airtable)
├── memoria_ALex.md               → Memoria principal del sistema
├── .env                          → Credenciales y variables de entorno
├── vps_deploy.sh                 → Script de deploy
└── venv/                         → Python virtualenv

/opt/geo-budget/                  → Geo Carpentry Budget Builder
/opt/geo-carpentry/               → Website Geo Carpentry
/opt/pinnacle-tools/              → Herramientas Pinnacle
```

---

## 3. SCRIPTS ACTIVOS Y SUS FUNCIONES

| Script | Ubicación | Función | Estado |
|--------|-----------|---------|--------|
| `alex_bot.py` | `/opt/alex-bot/telegram_bot/` | Bot Telegram — interfaz principal con Jorge | ✅ Running (systemd: `alex-bot.service`) |
| `el_polling.php` | Hostinger `/Tools/` | Cron cada 5min — skip trace de leads via Tracerfy | ✅ Activo |
| `el_chismoso.php` | Hostinger `/Tools/` | Webhook: recibe resultados Tracy → escribe en Airtable Contacts | ✅ Activo |
| `github_bridge.php` | Hostinger `/agents/` | Lee archivos de GitHub repos autorizados | ✅ HTTP 200 |
| `github_write.php` | Hostinger `/agents/` | Escribe/actualiza archivos en GitHub repos autorizados | ✅ Activo |
| `tracy_leads_poller.py` | `/opt/alex-bot/` | Poller Python alternativo para Tracy | Disponible |
| `tracy_skiptrace_automation.py` | `/opt/alex-bot/` | Automatización skip trace | Disponible |

---

## 4. ENDPOINTS PHP ACTIVOS EN agents.pinnaclegroupwi.com

### `GET` — Leer archivo de GitHub
```
GET https://agents.pinnaclegroupwi.com/github_bridge.php?repo={repo}&file={file}
Header: X-Alex-Secret: {ALEX_SECRET}
Repos permitidos: alex-real-estate-system, pinnacle-agent-memory, geo-budget-pro, pinnacle-tools, geo-carpentry
```

### `POST` — Escribir/actualizar archivo en GitHub
```
POST https://agents.pinnaclegroupwi.com/github_write.php
Header: X-Alex-Secret: {ALEX_SECRET}
Body: { "repo": "nombre", "file": "ruta/archivo.md", "content": "...", "message": "commit msg" }
```

### `GET/POST` — Skip Trace Cron
```
https://pinnaclegroupwi.com/Tools/el_polling.php   → corre manualmente o via cron
https://pinnaclegroupwi.com/Tools/el_chismoso.php  → webhook receptor
```

---

## 5. CONEXIÓN VPS ↔ HOSTINGER

```
VPS (187.77.215.146)
    ↕ HTTPS requests
agents.pinnaclegroupwi.com  (Hostinger)
    ↕ GitHub API (Bearer token en alex_config.php)
github.com/geocarp24/*
```

- El bot `alex_bot.py` llama a `github_bridge.php` para **leer** memoria desde GitHub
- El bot `alex_bot.py` llama a `github_write.php` para **escribir** memoria a GitHub
- Hostinger actúa como proxy autenticado entre el VPS y la GitHub API
- Autenticación: header `X-Alex-Secret` en todas las llamadas VPS→Hostinger

---

## 6. CONEXIÓN VPS ↔ GITHUB (directa)

```
~/.ssh/github  →  SSH key configurado para github.com
Host: github.com
IdentityFile: ~/.ssh/github
Repo: git@github.com:geocarp24/alex-real-estate-system.git
```

- Claude Code puede hacer `git pull`, `git push`, `git clone` directamente vía SSH
- No necesita pasar por Hostinger para operaciones git
- Repos autenticados: todos bajo `geocarp24/`

---

## 7. QUÉ PUEDE Y QUÉ NO PUEDE HACER CLAUDE CODE DESDE AQUÍ

### ✅ PUEDE:
- Leer y editar cualquier archivo en `/opt/alex-bot/` y subdirectorios
- Ejecutar comandos bash en el VPS
- Hacer git push/pull a GitHub via SSH
- Llamar APIs externas (Airtable, Telegram, Tracerfy, Make.com, Blotato) via curl/Python
- Invocar sub-agentes (El Scout, Matemático, Fact-Checker, Tracy) via Agent tool
- Leer/escribir memoria compartida (`memoria_ALex.md`, `telegram_memory.md`)
- Publicar en redes sociales via Blotato MCP (FB: Pinnacle Holdings Group, IG: @pinnacle.groupwi)
- Crear/actualizar registros en Airtable (CRM Real Estate + Social Media)
- Reiniciar el bot de Telegram via systemctl

### ❌ NO PUEDE (sin acción manual de Jorge):
- Acceder a la UI de Hostinger o cPanel directamente
- Ejecutar código PHP localmente (sin servidor web)
- Autenticarse en Make.com, Canva, Cloudinary sin OAuth manual
- Recibir webhooks entrantes sin un servidor HTTP expuesto
- Ser invocado por el bot de Telegram automáticamente (aún — ver sección de puente abajo)

---

## 8. SISTEMAS MCP DISPONIBLES EN ESTA SESIÓN

| MCP | Tools disponibles |
|-----|------------------|
| **Blotato** | create_post, list_accounts, create_source, create_visual, list_schedules, etc. |

---

*Generado por ALEX Claude Code — 2026-04-05*
