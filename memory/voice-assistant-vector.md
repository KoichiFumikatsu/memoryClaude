# Voice Assistant "Vector"

Asistente de voz personal en `/home/kelsie/voice-assistant/`. Operativo desde 2026-05-08. Renombrado a Vector y extendido con modo Claude Code el 2026-05-25.

## Wake word y arquitectura

- **Keyword: "vector"** (antes "sistema")
- Daemon: `vector.py` (antes `daemon.py`, archivado como `.old.bak`)
- Servicio systemd user: `vector.service` con autoarranque al login, logs en `vector.log`, requiere `PYTHONUNBUFFERED=1`
- Notificador separado: `voice-notifier.service` (no usa mic, no choca)

## Dos modos de respuesta

Estado persistido en `~/voice-assistant/.mode`:

- **chat** (default): Claude Haiku 4.5 vía Anthropic API, respuesta corta hablada
- **code**: Claude Code CLI (`claude-sonnet-4-6`) con sesión persistente vía `--resume <uuid>`, respuesta hablada por TTS

UUID de la sesión en `~/voice-assistant/.claude_session_id`. Primera llamada usa `--session-id`, siguientes `--resume`.

## Comandos de control por voz

- "cambiar a modo código" / "modo código" → switch a code
- "cambiar a modo chat" / "modo haiku" → switch a chat
- "reiniciar sesión" / "nueva conversación" → borra session_id
- "qué modo estás" → reporta modo

Reset chequeado antes de switch_chat para que "nueva conversación" no caiga mal.

## Configuración modo code

- cwd: `/home/kelsie`
- model: `claude-sonnet-4-6`
- permission_mode: `bypassPermissions` (justificado: por voz no se pueden confirmar prompts)
- timeout: 120s
- output-format: json, se extrae el campo `result`

## Stack adicional

- STT: faster-whisper small (CPU, local) tanto para keyword como para comando
- TTS: gTTS + mpg123
- Clima: Open-Meteo (sin key)
- Noticias: RSS El Tiempo Cali + Google News
- Calendar: Google Calendar API v3 OAuth2
- Push celular: ntfy.sh topic `koichi_agenda_2026`

## Decisiones de diseño tomadas

- Keyword spotter con Whisper en lugar de OpenWakeWord (acento colombiano)
- Selección de modo por voz con estado persistente (no prefijo por comando)
- Sesión persistente en modo code (no calls aisladas)
- bypassPermissions en code mode (sin esto la voz se rompe en prompts)
- Briefing solo muestra primer evento; el resto los maneja notifier
- Fines de semana: briefing a las 8:00am fijo

## Pendiente

- Wake word personalizado con OpenWakeWord
- Mejorar STT para comandos largos
- Truncado de respuestas largas en modo code (gTTS lee todo)
- Fase 3: migrar a RPi o celular vía FastAPI
