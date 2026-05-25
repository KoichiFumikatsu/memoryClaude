---
name: Meeting Recorder — Grabacion + Transcripcion + Resumen
description: Sistema local de grabacion de reuniones presenciales con Whisper + Ollama, instalado 2026-05-25
type: project
---

# Meeting Recorder — Fumilinux

Instalado 2026-05-25. Flujo completo para reuniones presenciales con limitacion de escritura en una mano.

## Flujo principal

1. Abrir **"Grabar Reunion"** desde Activities (GNOME launcher)
2. Responder preguntas: idioma, modelo Whisper, si transcribir automaticamente, si resumir
3. Grabar — boton "Detener reunion" para parar
4. Transcripcion con faster-whisper (post-meeting, mucho mas preciso que tiempo real)
5. Resumen con Ollama enfocado en Departamento de Sistemas

## Comandos instalados en PATH

| Comando | Funcion |
|---|---|
| `grabar-reunion` | CLI version del grabador |
| `transcribir-reunion` | Transcribe WAV con faster-whisper + resumen Ollama |
| `resumir` | Toma texto del portapapeles y genera resumen con Ollama |
| `dictar` | Nerd Dictation — dictado en tiempo real via Vosk (español) |
| `dictar-stop` | Detiene Nerd Dictation |

## Archivos

- Scripts: `/home/kelsie/reuniones/`
- Grabaciones: `/home/kelsie/reuniones/grabaciones/`
- Transcripciones y resumenes: `/home/kelsie/reuniones/transcripts/`
- GNOME launcher: `~/.local/share/applications/grabar-reunion.desktop`
- Nerd Dictation: `/home/kelsie/nerd-dictation/` — modelo ES: `model/es/`

## Stack instalado

- `faster-whisper` — transcripcion post-meeting (modelos: small/medium/large-v3)
- `vosk` + `nerd-dictation` — dictado en tiempo real sin alucinaciones
- `sounddevice`, `numpy` — dependencias de audio Python

## Audio

- Fuente por defecto cambiada a `easyeffects_source` (noise cancelled)
- `pw-record` para grabacion via PipeWire
- Ollama modelo: `llama3.2:3b` en `localhost:11434`

## Por que grabacion post-meeting y no tiempo real

Whisper alucina en tiempo real con ruido de sala y voces multiples. Sobre archivo completo es mucho mas preciso. RealtimeSTT, webrtcvad, Chromium y Chrome fueron probados y descartados en esta sesion.
