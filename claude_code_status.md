# CLAUDE CODE STATUS — Sistema ALEX
> Última actualización: 2026-04-05 | VPS: 187.77.215.146

---

## SERVICIOS ACTIVOS (4 servicios)

| Servicio | Puerto | Estado | Función |
|---------|--------|--------|--------|
| `alex-bot.service` | — | ✅ Running | Bot Telegram — interfaz con Jorge |
| `claude-worker.service` | — | ✅ Running | Worker file-based — fallback local |
| `claude-api.service` | 5001 | ✅ Running | HTTP API Server — puente directo Bot↔Claude Code |
| `github-monitor.service` | — | ✅ Running | **Monitor 24/7 — lee task_queue.json en GitHub cada 30s** |

---

## FLUJO COMPLETO — Jorge nunca más abre Claude Code

```
Jorge escribe en Telegram: /tarea <descripción>
    ↓
ALEX Bot escribe en GitHub: task_queue.json {status: pendiente}
    ↓
github-monitor (VPS, cada 30s) detecta tarea pendiente
    ↓
Cambia status → en_proceso (escribe en GitHub)
    ↓
Ejecuta: claude --print "<tarea>" (Claude Code CLI)
    ↓
Cambia status → completado + result (escribe en GitHub)
    ↓
Escribe resumen en memoria_ALex.md
    ↓
Envía resultado directo a Jorge en Telegram
    ✅ COMPLETADO
```

---

## COMANDOS TELEGRAM DISPONIBLES

| Comando | Función |
|---------|--------|
| `/tarea <desc>` | Escribe en GitHub queue → Monitor ejecuta → Telegram |
| `/claude <desc>` | HTTP API local (puerto 5001) → Claude Code → Telegram |
| `/start` | Saludo de ALEX |
| `/reset` | Limpiar historial |
| `/guardar` | Guardar memoria |
| `/memoria` | Ver memoria |

---

## ENDPOINTS HTTP (claude_api_server.py :5001)

```
POST http://localhost:5001/task          → crear tarea
GET  http://localhost:5001/task/<id>     → consultar resultado
GET  http://localhost:5001/health        → health check
GET  http://localhost:5001/status        → métricas
Auth: X-Alex-Secret header requerido
```

---

## ARCHIVOS DEL SISTEMA

```
/opt/alex-bot/
├── github_monitor.py          → Monitor GitHub 24/7 (NUEVO)
├── claude_api_server.py       → HTTP API Server :5001
├── claude_worker.py           → Worker file-based (fallback)
├── telegram_bot/alex_bot.py   → Bot con /tarea y /claude
├── agents/
│   ├── task_queue.json        → Cola local (fallback)
│   ├── shared_conversation.json → Espejo Telegram↔Claude Code
│   ├── github_monitor.log     → Log del monitor
│   └── claude_api_server.log  → Log del API server

geocar24/pinnacle-agent-memory (GitHub)
├── task_queue.json            → COLA PRINCIPAL — Monitor la lee cada 30s
├── cola_mensajes.md           → Log humano-legible
└── claude_code_status.md      → Este archivo
```

---

## TEST E2E — PASADO ✅ 2026-04-05

```
Tarea inyectada en task_queue.json: status=pendiente
Monitor detectó en ~13 segundos
Ejecutó Claude Code CLI
Resultado: MONITOR GITHUB ACTIVO - Sistema de monitoreo 24/7 funcionando.
Status actualizado: completado
Resultado enviado a Telegram: ✅
Tiempo total: ~10 segundos de ejecución
```

---

## COMANDOS DE ADMINISTRACIÓN

```bash
# Ver estado de todos los servicios
systemctl status alex-bot claude-api claude-worker github-monitor

# Ver logs en tiempo real
journalctl -u github-monitor.service -f
journalctl -u claude-api.service -f

# Reiniciar monitor
systemctl restart github-monitor.service

# Ver cola de tareas en GitHub
curl -s "https://agents.pinnaclegroupwi.com/github_bridge.php?repo=pinnacle-agent-memory&file=task_queue.json" \
  -H "X-Alex-Secret: $ALEX_SECRET"
```

---

*Generado por ALEX Claude Code — 2026-04-05*