# Claude Memory System

## Session Initialization

**PRIMERO — antes de leer cualquier archivo:** ejecuta git pull para tener el contexto más reciente de cualquier sesión anterior en otra máquina:
```
cd ~/memoryClaude-main && git pull origin main 2>/dev/null || true
```

At the start of every session, read the following files before doing anything else:

- [memory/personality.md](memory/personality.md) — **read first** — defines how to think, communicate, and make decisions for this user
- [memory/user.md](memory/user.md) — who the user is, their role, expertise, and goals
- [memory/preferences.md](memory/preferences.md) — how the user wants Claude to behave
- [memory/decisions.md](memory/decisions.md) — past decisions so they don't get re-litigated
- [memory/people.md](memory/people.md) — collaborators and contacts mentioned across sessions

## TL Games — Archivos adicionales

Si el contexto de la sesión involucra traducción de juegos, leer además:

- [memory/tl-traduccion-juegos-brief.md](memory/tl-traduccion-juegos-brief.md) — brief general del workspace
- [memory/tl-es-style.md](memory/tl-es-style.md) — guía de estilo español para traducciones
- [memory/tl-decisions.md](memory/tl-decisions.md) — historial técnico de decisiones por juego
- [memory/tl-refactor-5-stages.md](memory/tl-refactor-5-stages.md) — refactor pipeline_server a 5 etapas (estructura del flujo actual, archivos clave, settings, dashboard v2)

Según el engine activo:
- Ren'Py → [memory/tl-playbook-renpy.md](memory/tl-playbook-renpy.md)
  - Si detecta `kps_build_conversation_list` o `© Kesash` en `phone.rpy` → además [memory/tl-playbook-kps-phone.md](memory/tl-playbook-kps-phone.md)
- Unity → [memory/tl-unity-brief.md](memory/tl-unity-brief.md) + [memory/tl-playbook-unity.md](memory/tl-playbook-unity.md)
- RPG Maker → [memory/tl-rpgmaker-brief.md](memory/tl-rpgmaker-brief.md) + [memory/tl-playbook-rpgmaker.md](memory/tl-playbook-rpgmaker.md)
- GameMaker → [memory/tl-gamemaker-brief.md](memory/tl-gamemaker-brief.md) + [memory/tl-playbook-gamemaker.md](memory/tl-playbook-gamemaker.md)

## AI Image Generation

Si el contexto involucra generación de imágenes AI (SD.Next, Stable Diffusion, anime/SDXL/SD1.5, instalación en Torre 1), leer:
- [memory/ai-image-local.md](memory/ai-image-local.md) — Estado: REMOVIDO de Fumilinux 2026-05-24. Plan migrar a Torre 1 RX 570. Aprendizajes técnicos preservados: trampas OpenVINO FX, URLs de modelos, prompts probados, métricas de comparación

## Torre 1 — Linux + Mirror

Si el contexto involucra instalar Linux en Torre 1 o replicar el entorno de Fumilinux, leer:
- [memory/torre1-linux-mirror.md](memory/torre1-linux-mirror.md) — Torre 1 completamente clonada desde Fumilinux (2026-05-26). Usuario `kelsielinux`, IP 192.168.12.7. ROCm activo. Pendiente: rclone OAuth, n8n workflows, Vector venv.

## Kelsie TL — Brand Project

Si el contexto involucra la marca pública Kelsie TL (identidad visual, redes sociales, web, releases), leer:
- [memory/kelsietl-brand.md](memory/kelsietl-brand.md) — sistema visual completo, assets Figma, estado del proyecto, decisiones tomadas

## Memorias específicas por proyecto/refactor

Cuando un proyecto tiene un refactor mayor, decisiones arquitectónicas extensas o documentación de un flujo completo que excede `decisions.md`, crear un archivo dedicado `<proyecto>-<tema>.md` en `memory/` y enlazarlo aquí. Reglas:
- El archivo dedicado se carga solo cuando el contexto lo amerite (mencionado por el usuario, cwd dentro del proyecto, o trabajando sobre sus archivos).
- Mantener `decisions.md` con resumen de una línea apuntando al archivo dedicado.
- Replicar el archivo en `/home/kelsie/.claude/projects/-home-kelsie/memory/` con frontmatter + indexarlo en `MEMORY.md`.

---

## Session End

Before finishing any session, update the relevant memory files with:
- New facts learned about the user → `memory/user.md`
- New preferences or corrections given → `memory/preferences.md`
- Important decisions taken → `memory/decisions.md`
- New people mentioned → `memory/people.md`

Then the stop hook will auto-commit and push any changes.

## Memory Update Rules

- Only write what is **non-obvious** and **reusable** across sessions.
- Do not store ephemeral task state (use TodoWrite for that).
- Do not store what is already in git history or the code itself.
- Always convert relative dates to absolute dates (e.g. "Thursday" → "2026-05-08").
- When a memory conflicts with current reality, update the file — don't act on stale data.
