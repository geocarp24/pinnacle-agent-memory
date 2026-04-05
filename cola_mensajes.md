# COLA DE MENSAJES — ALEX Bot ↔ Claude Code

> Canal de comunicación bidireccional entre ALEX Telegram Bot (VPS) y Claude Code (CLI).
> Última actualización: 2026-04-05

---

## ESTADO DEL SISTEMA

| Componente | Estado | Ubicación |
|-----------|--------|-----------|
| ALEX Telegram Bot | ✅ Activo | VPS 187.77.215.146, systemd |
| Claude Code CLI | ✅ Disponible | VPS 187.77.215.146, /usr/local/bin/claude |
| Bridge Hostinger | ✅ HTTP 200 | agents.pinnaclegroupwi.com |
| GitHub (puente de estado) | ✅ Activo | geocarp24/pinnacle-agent-memory |

---

## ARQUITECTURA DEL PUENTE BOT ↔ CLAUDE CODE

### Problema actual
El Bot de Telegram corre como proceso systemd (`alex-bot.service`). Claude Code corre como proceso interactivo separado. No tienen comunicación directa — Jorge tiene que copiar y pegar resultados entre ambos.

### Solución propuesta: Puente via archivos + systemd

```
Jorge (Telegram)
    ↓ mensaje: "/claude <tarea>"
ALEX Bot (alex_bot.py)
    ↓ escribe tarea en /opt/alex-bot/agents/claude_inbox.json
    ↓ ejecuta: claude --print "<prompt>" en subprocess
Claude Code CLI
    ↓ procesa la tarea
    ↓ escribe resultado en /opt/alex-bot/agents/claude_outbox.json
ALEX Bot
    ↓ lee claude_outbox.json
    ↓ envía respuesta a Jorge via Telegram
```

### Mecanismo de implementación

**Opción A — subprocess directo (recomendada para tareas rápidas):**
```python
import subprocess
result = subprocess.run(
    ["claude", "--print", prompt],
    capture_output=True, text=True, timeout=120,
    cwd="/opt/alex-bot"
)
response = result.stdout
```

**Opción B — archivo de cola (para tareas largas o asíncronas):**
```
/opt/alex-bot/agents/
├── claude_inbox.json    → Bot escribe tareas aquí
├── claude_outbox.json   → Claude Code escribe resultados aquí
└── claude_status.json   → Estado: idle | processing | done | error
```

Un watcher script (`claude_worker.py`) corre en background y procesa el inbox continuamente.

**Opción C — GitHub como cola (sin dependencia de procesos locales):**
```
Bot escribe tarea → este archivo (cola_mensajes.md) en GitHub
Claude Code lee → procesa → escribe resultado en GitHub
Bot lee resultado → responde a Jorge
```
Esta opción ya funciona parcialmente con el bridge existente.

---

## PLAN DE IMPLEMENTACIÓN — PUENTE DIRECTO

### FASE 1: subprocess simple (1-2 horas de trabajo)

**Archivos a modificar:**
- `/opt/alex-bot/telegram_bot/alex_bot.py` — agregar comando `/claude`

**Lógica a agregar en alex_bot.py:**
```python
@bot.message_handler(commands=['claude'])
def handle_claude_task(message):
    task = message.text.replace('/claude ', '', 1)
    bot.send_message(message.chat.id, "⏳ Claude Code procesando...")
    
    result = subprocess.run(
        ["claude", "--print", task],
        capture_output=True, text=True, timeout=180,
        cwd="/opt/alex-bot",
        env={**os.environ, "ANTHROPIC_API_KEY": os.getenv("ANTHROPIC_API_KEY")}
    )
    
    response = result.stdout if result.returncode == 0 else f"Error: {result.stderr}"
    bot.send_message(message.chat.id, response[:4000])  # Telegram limit
```

**Resultado:** Jorge escribe `/claude <tarea>` en Telegram → ALEX Bot llama a Claude Code → respuesta en Telegram.

---

### FASE 2: Cola asíncrona con watcher (3-4 horas)

**Archivos nuevos:**
- `/opt/alex-bot/claude_worker.py` — daemon que procesa cola
- `/etc/systemd/system/claude-worker.service` — servicio systemd
- `/opt/alex-bot/agents/claude_inbox.json` — cola de entrada
- `/opt/alex-bot/agents/claude_outbox.json` — cola de salida

**Flujo:**
1. Bot escribe en `claude_inbox.json`: `{"task_id": "uuid", "prompt": "...", "chat_id": 123}`
2. `claude_worker.py` detecta nueva tarea → ejecuta Claude Code → escribe en `claude_outbox.json`
3. Bot monitorea `claude_outbox.json` → envía resultado a Telegram

**Ventaja:** Claude Code puede procesar tareas largas sin timeout de Telegram.

---

### FASE 3: Bidireccional completo (futuro)

Claude Code también puede **iniciar** conversaciones con Jorge via Telegram:
```python
# Desde Claude Code / sub-agentes:
import requests
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
JORGE_CHAT_ID = os.getenv("TELEGRAM_CHAT_ID")
requests.post(f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage",
    json={"chat_id": JORGE_CHAT_ID, "text": mensaje})
```

Ya existe `alerta_telegram.sh` que hace exactamente esto — puede usarse como base.

---

## PRIORIDAD DE IMPLEMENTACIÓN

| Fase | Esfuerzo | Impacto | Prioridad |
|------|---------|---------|-----------|
| Fase 1 — subprocess `/claude` | Bajo (1-2h) | Alto | 🔴 PRIMERO |
| Fase 2 — cola asíncrona | Medio (3-4h) | Alto | 🟡 SEGUNDO |
| Fase 3 — Claude inicia Telegram | Bajo (30min) | Medio | 🟢 TERCERO |

---

## MENSAJES ACTIVOS EN COLA

*(Vacío — sistema inicializado 2026-04-05)*

---

## HISTORIAL

| Fecha | Mensaje | Estado |
|-------|---------|--------|
| 2026-04-05 | Sistema documentado. Puente propuesto. | ✅ |

---

*Generado por ALEX Claude Code — 2026-04-05*
