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

## AI Image / Video Generation

Si el contexto involucra generación de imágenes o video AI (Stable Diffusion, Pony, SDXL, anime, AnimateDiff, SVD, instalación en Torre 1, `~/projects/ia-gen/`), leer:
- [memory/ai-image-local.md](memory/ai-image-local.md) — Estado activo: sdcpp + Vulkan + PonyDiffusion V6 XL en Torre Oficina. Workspace en `~/projects/ia-gen/`. Open WebUI con OpenAI/Groq/Gemini pre-configurado. Aprendizajes: trampas Vulkan, OOM VAE, métricas, prompts Pony.
- [memory/hardware-3-torres.md](memory/hardware-3-torres.md) — inventario completo Oficina/Casa/AZC, plan swap CPU+RAM Casa↔Oficina, futuro upgrade GPU. Leer cuando se discuta hardware, upgrades, swaps o GPU.

## Torre 1 — Linux + Mirror

Si el contexto involucra instalar Linux en Torre 1 o replicar el entorno de Fumilinux, leer:
- [memory/torre1-linux-mirror.md](memory/torre1-linux-mirror.md) — Torre 1 completamente clonada desde Fumilinux (2026-05-26). Usuario `kelsielinux`, IP 192.168.12.7. ROCm activo. Pendiente: rclone OAuth, n8n workflows, Vector venv.

## Kelsie TL — Brand Project

Si el contexto involucra la marca pública Kelsie TL (identidad visual, redes sociales, web, releases), leer:
- [memory/kelsietl-brand.md](memory/kelsietl-brand.md) — sistema visual completo, assets Figma, estado del proyecto, decisiones tomadas

## dashlife — Finanzas personales

Si la sesión involucra dashlife (PWA de finanzas, cwd `/home/kelsie/projects/dashlife`, o se menciona el cuadre de extracto / cobros fijos / KPIs), leer:
- [memory/project_dashlife.md](memory/project_dashlife.md) — Rebuild de KelsieApp (Fase 1 finanzas). SvelteKit+SQLite+Drizzle+Claude Haiku+ntfy, self-host Fumilinux/Tailscale :8443. Despliegue systemd, cuadre de extracto (CSV/foto Nu vía Claude visión), KPIs explicativos, cobro fijo=pago de tarjeta con aviso un día antes, offline outbox. Trampas: BODY_SIZE_LIMIT=15M, COP usa "." de miles en Claude visión, each-key duplicado rompe la lista.

## Troncal SIP TIGO + UCM6308 (telefonía AZC)

Si la sesión involucra el troncal SIP de TIGO, el PBX Grandstream UCM6308, el router TP-Link ER605, troncales DIDWW, o problemas de audio/registro/NAT en telefonía AZC, leer:
- [memory/troncal-tigo-ucm-er605.md](memory/troncal-tigo-ucm-er605.md) — Config completa del troncal TIGO (enlace /30 `172.24.249.66`, SIP server `172.17.179.150`, auth por IP, sin registro) detrás del ER605, sin romper DIDWW ni internet. Apuntamiento: UCM→`172.17.179.150` / ruta estática ER605→`172.24.249.65` / NAT salida `172.24.249.66`. Estado (2026-06-24): internet ✅, DNS ✅, internacional con audio ✅, **TIGO audio en una sola vía ⚠️** (UCM anuncia IP privada en SDP, TIGO no hace latching). Pendiente: pedir Symmetric RTP a TIGO o pasar TIGO a 2º puerto del UCM; y Wave WebRTC. Gotchas: typo next-hop, DNS de TIGO rompe resolución pública, Link Backup deja WAN en standby, SIP ALG global = conflicto TIGO↔internet.

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
