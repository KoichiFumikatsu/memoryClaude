# Claude Memory System

## Session Initialization

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

Según el engine activo:
- Ren'Py → [memory/tl-playbook-renpy.md](memory/tl-playbook-renpy.md)
- Unity → [memory/tl-unity-brief.md](memory/tl-unity-brief.md) + [memory/tl-playbook-unity.md](memory/tl-playbook-unity.md)
- RPG Maker → [memory/tl-rpgmaker-brief.md](memory/tl-rpgmaker-brief.md) + [memory/tl-playbook-rpgmaker.md](memory/tl-playbook-rpgmaker.md)
- GameMaker → [memory/tl-gamemaker-brief.md](memory/tl-gamemaker-brief.md) + [memory/tl-playbook-gamemaker.md](memory/tl-playbook-gamemaker.md)

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
