# IA Gen — Dashboard de generación (Torre 1) — bug watchdog y arquitectura de chains

**Actualizado 2026-06-08.**

Generación local de imágenes anime SDXL (modelo `wai`=waiNSFWIllustrious y otros) en **Torre 1** (RX 570, `kelsielinux@100.67.216.43`, sd-server stable-diffusion.cpp :7860). El **dashboard corre en Fumilinux** como servicio systemd de usuario `iagen-dashboard` (puerto 8769), orquesta todo por SSH.

- Repo local: `/home/kelsie/projects/ia-gen` — `dashboard.py` (~2833 líneas), `velvet/forge.py`, `chains/`, `scripts/`.
- Tras editar `dashboard.py` o `velvet/forge.py`: `systemctl --user restart iagen-dashboard` (Python cachea `import forge`; sin restart sigue la versión vieja).

## Arquitectura de "chains"
Toda generación de ≥2 imágenes va como script bash lanzado desacoplado en Torre 1
(`setsid bash -c 'nohup ./chain_X.sh > /tmp/chain_X.out 2>&1' </dev/null & disown`).
El dashboard detecta "chain activa" con `pgrep -af 'gen\.sh|chain_.*\.sh|orchestrator.*\.sh'`
y parsea el header `──── [N/TOTAL] label seed S | HH:MM:SS ────`. Si hay chain activa,
bloquea nuevos lanzamientos con "Torre 1 ocupada".

Dos generadores (en proceso de unificación a un solo modal, 2026-06-08):
- **Cajón** (`run_generate` en dashboard.py): chars × seeds de una composición. Arma la chain inline.
- **Velvet Forge** (`run_velvet_forge` → `velvet/forge.py:build_chain`): set por personaje con tiers
  SFW/sugerente/NSFW de velvet.json. Koichi lo va a renombrar (lo usa para chain de múltiples
  composiciones de UN personaje) y agregar composiciones multi-personaje. Salida pasará a la
  misma carpeta del cajón (antes `velvet/<guest_id>`).

## BUG CRÍTICO resuelto (2026-06-08): chain colgada en wait() por watchdog no muerto
**Síntoma:** chain de Velvet "nunca finalizaba" aunque las imágenes ya estaban hechas y
sincronizadas; el dashboard reportaba "Torre 1 ocupada" para siempre.
**Causa raíz:** el bash de la chain quedaba en `STAT=S / WCHAN=do_wait`, bloqueado en `wait()`
esperando al **watchdog** (loop infinito) que ella misma lanzaba y **nunca mataba**. El watchdog
nunca termina → bash nunca sale → `pgrep 'chain_.*\.sh'` lo ve vivo eternamente.
**Por qué el cajón NO lo tiene:** su última instrucción mata el watchdog:
`for p in $(pgrep -f '[w]atchdog.sh'); do kill "$p" 2>/dev/null; done`
**Fix:** `forge.py:build_chain()` ahora añade esa misma línea como paso final.
**Regla general:** toda chain que lance un watchdog DEBE matarlo como último paso.
**Finalizar una chain ya colgada:** matar el watchdog por PID vía `pgrep -f '[w]atchdog.sh'`
(NUNCA `pkill -f` → se auto-mata el shell). El proceso de la chain sale solo.
