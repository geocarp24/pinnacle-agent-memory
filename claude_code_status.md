# CLAUDE CODE STATUS — Sistema ALEX
> Última actualización: 2026-04-05 | Servidor VPS: 187.77.215.146

---

## 1. SERVICIOS ACTIVOS EN EL VPS

| Servicio | Puerto | Estado | Función |
|---------|--------|--------|---------|
| `alex-bot.service` | — | ✅ Running | Bot Telegram — interfaz con Jorge |
| `claude-worker.service` | — | ✅ Running | Worker file-based — procesa claude_inbox.json |
| `claude-api.service` | 5001 | ✅ Running | **HTTP API Server — puente directo Bot↔Claude Code** |

---

## 2. ARQUITECTURA DEL PUENTE (IMPLEMENTADA)

```
Jorge (Telegram)
    ↓ /claude <tarea>
ALEX Bot (alex_bot.py)
    ↓ POST http://localhost:5001/task
    ↓ Header: X-Alex-Secret
Claude API Server (claude_api_server.py :5001)
    ↓ encola tarea internamente (threading.Queue)
    ↓ worker thread ejecuta: claude --print "<historial + tarea>"
    ↓ envía resultado directo a Telegram via Bot API
Jorge (Telegram)
    ✅ recibe respuesta automáticamente

Fallback si API no responde:
    → escribe en claude_inbox.json
    → claude_worker.py lo procesa (polling cada 2s)
```

---

## 3. ENDPOINTS HTTP — claude_api_server.py

### `POST /task` — Crear tarea
```
URL:    http://localhost:5001/task  (o http://187.77.215.146:5001/task externamente)
Header: X-Alex-Secret: <ALEX_SECRET>
Body:   {"prompt": "...", "chat_id": "...", "source": "telegram|claude_code|..."}
Resp:   {"task_id": "uuid", "status": "pending", "message": "..."}
HTTP:   202 Accepted
```

### `GET /task/<task_id>` — Consultar resultado
```
URL:    http://localhost:5001/task/<uuid>
Header: X-Alex-Secret: <ALEX_SECRET>
Resp:   {"task_id", "status", "response", "success", "completed_at", ...}
```

### `GET /health` — Health check
```
URL:    http://localhost:5001/health
Resp:   {"status":"ok", "queue": N, "tasks": N, "timestamp": "..."}
```

### `GET /status` — Estado del sistema
```
URL:    http://localhost:5001/status
Header: X-Alex-Secret: <ALEX_SECRET>
Resp:   {tasks_total, tasks_done, tasks_error, tasks_pending, ...}
```

---

## 4. ARCHIVOS DEL SISTEMA

```
/opt/alex-bot/
├── claude_api_server.py          → HTTP API Server (Flask, puerto 5001)
├── claude_worker.py              → Worker daemon (file-based, fallback)
├── telegram_bot/alex_bot.py      → Bot Telegram — usa API server para /claude
├── agents/
│   ├── claude_inbox.json         → Cola file-based (worker fallback)
│   ├── claude_outbox.json        → Resultados file-based
│   ├── shared_conversation.json  → Espejo conversación Telegram↔Claude Code
│   ├── claude_api_server.log     → Log del API server
│   └── claude_worker.log         → Log del worker
```

---

## 5. SERVICIOS SYSTEMD

```bash
# Ver estado
systemctl status claude-api.service
systemctl status claude-worker.service
systemctl status alex-bot.service

# Reiniciar
systemctl restart claude-api.service

# Logs
journalctl -u claude-api.service -f
journalctl -u claude-worker.service -f
```

---

## 6. AUTO-DETECCIÓN EN EL BOT

El bot detecta automáticamente mensajes que contienen estas palabras clave
y los puede delegar a Claude Code:
- ejecuta, corre el script, bash, shell, systemctl
- git commit, git push, deploy, instala, pip install
- edita el archivo, modifica el código, actualiza el bot
- reinicia el servicio, lee el log, muestra los logs

---

## 7. TEST E2E — PASADO ✅

```
POST /task → {"prompt": "API BRIDGE ACTIVO..."}
→ status: done
→ response: "API BRIDGE ACTIVO - Puente HTTP Bot↔Claude Code funcionando al 100%."
→ Resultado enviado a Telegram: ✅
```
Tiempo de procesamiento: ~5 segundos

---

## 8. QUÉ PUEDE HACER EL SISTEMA AHORA

| Capacidad | Método |
|-----------|--------|
| Jorge escribe `/claude <tarea>` en Telegram | → API server ejecuta Claude Code → resultado en Telegram |
| Claude Code inicia mensaje a Jorge | → `send_telegram()` directo desde worker/API |
| Contexto compartido entre Telegram y Claude Code | → `shared_conversation.json` (espejo) |
| Fallback si API cae | → `claude_inbox.json` + worker daemon |
| Consultar estado de tarea | → `GET /task/<id>` |
| Sistema externo (Hostinger, Make.com) envía tarea | → `POST /task` con X-Alex-Secret |

---

*Actualizado por ALEX Claude Code — 2026-04-05*
