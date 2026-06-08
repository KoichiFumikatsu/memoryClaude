---
name: project_claude_skills_plugins
description: "Skills y plugins de Claude Code instalados globalmente (find-skills, web-design-guidelines, agent-browser, frontend-design, superpowers) y cómo gestionarlos"
metadata: 
  node_type: memory
  type: project
  originSessionId: dc364104-1eb1-4ebd-87ef-7be537230f0a
---

Instalados 2026-06-08 (sesión Koichi). Cargan al **iniciar sesión nueva**, no en la sesión donde se instalan.

**Skills** (en `~/.claude/skills/`, gestor `npx skills@latest`):
- `find-skills` ← vercel-labs/skills
- `web-design-guidelines` ← vercel-labs/agent-skills
- `agent-browser` ← vercel-labs/agent-browser (necesita binario CLI aparte)

Comandos: `npx skills add <repo> -g -a claude-code -s <skill> -y` (global, sin sudo, copia a `~/.claude/skills/`). Actualizar: `npx skills update -g`. Listar: `npx skills ls`.

**Plugins** (vía CLI `claude plugin install`, scope user, enabled):
- `frontend-design@claude-plugins-official` — el marketplace oficial ya estaba clonado; se prefirió plugin sobre duplicar la SKILL.md.
- `superpowers@superpowers-marketplace` v5.1.0 — marketplace `obra/superpowers-marketplace` agregado vía `claude plugin marketplace add` (clona por SSH).

**Binario CLI** `agent-browser` v0.27.0 en `/usr/bin/` (instalado con `sudo npm install -g agent-browser`; npm prefix=`/usr` no escribible → requiere sudo). Chrome for Testing 149 en `~/.agent-browser/browsers/`. Si falla por "shared library": `agent-browser install --with-deps`.

Plugin preexistente no relacionado: `ui-ux-pro-max@nextlevelbuilder`. Ver [[feedback_sudo]].
