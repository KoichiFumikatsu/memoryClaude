# Claude Code launchers — Fumilinux

Creados 2026-05-25. Espejo de `~/.claude/projects/-home-kelsie/memory/project_claude_launchers.md` (regla persistencia dual).

## Ubicación

`~/.local/share/applications/claude-*.desktop` (solo menú de apps GNOME, no escritorio).

## Lanzadores

| Archivo | Proyecto | cwd | --permission-mode |
|---|---|---|---|
| `claude-tlgames.desktop` | TL Games | `/home/kelsie/projects/tlgames` | `acceptEdits` |
| `claude-companero.desktop` | Compañero | `/home/kelsie/projects/companero` | `plan` |
| `claude-voice-assistant.desktop` | Voice Assistant | `/home/kelsie/voice-assistant` | (default — sin flag) |
| `claude-azckeeper.desktop` | AZCKeeper | `/home/kelsie/projects/azckeeper` | `plan` |
| `claude-kelsieapp.desktop` | KelsieApp | `/home/kelsie/projects/kelsie-app` | `plan` |

Modelo siempre `claude-opus-4-7`.

## Patrón Exec

```
gnome-terminal --title="Claude <Proyecto>" --working-directory=<path> -- bash -c "claude --model claude-opus-4-7 [--permission-mode <mode>] --name '<Proyecto>'; exec bash"
```

- `; exec bash` mantiene el terminal abierto al cerrar Claude.
- `--name` etiqueta la sesión en prompt box y `/resume`.
- Icon=`utilities-terminal` (genérico — reemplazar por icono custom si se desea).

## Carpetas placeholder creadas

- `/home/kelsie/projects/azckeeper/` con `CLAUDE.md` mínimo (apunta a memoria global; el código vive en `keep.azclegal.com`)
- `/home/kelsie/projects/kelsie-app/` con `CLAUDE.md` mínimo (apunta a memoria global; producción en Vercel)

## Trampas (no reintroducir)

- **`/init` NO carga contexto** — crea un `CLAUDE.md` nuevo. El contexto se carga automáticamente cuando `claude` arranca con cwd en el proyecto. No incluir slash commands iniciales en el `Exec=`.
- **Modos válidos** (`claude --help` 2026-05-25): `acceptEdits`, `auto`, `bypassPermissions`, `default`, `dontAsk`, `plan`.
- **Modelo IDs vigentes**: `claude-opus-4-7`, `claude-sonnet-4-6`, `claude-haiku-4-5-20251001`. Alias `opus`/`sonnet`/`haiku` también funcionan.
- Tras crear o editar `.desktop`: `chmod +x` + `update-desktop-database ~/.local/share/applications/`.

## Añadir uno nuevo

1. `cp ~/.local/share/applications/claude-tlgames.desktop ~/.local/share/applications/claude-<nuevo>.desktop`
2. Editar `Name=`, `Comment=`, `--title`, `--working-directory`, `--permission-mode`, `--name`
3. `chmod +x` y `update-desktop-database ~/.local/share/applications/`
