# IA Gen — Dashboard de generación (Torre 1) — bug watchdog y arquitectura de chains

**Actualizado 2026-06-26** (estado de código verificado contra el repo; última edición real de `dashboard.py` 2026-06-22).

Generación local de imágenes anime SDXL (modelo `wai`=waiNSFWIllustrious y otros) en **Torre 1** (RX 570 **4GB**, `kelsielinux@100.67.216.43`, sd-server stable-diffusion.cpp :7860). El **dashboard corre en Fumilinux** como servicio systemd de usuario `iagen-dashboard` (puerto 8769), orquesta todo por SSH.

- Repo local: `/home/kelsie/projects/ia-gen` — `dashboard.py` (**~5335 líneas** a 2026-06-22; eran ~2833 el 06-14), `velvet/forge.py` (248), `chains/`, `scripts/`, `docs/`.
- **NO es repo git** — solo backups manuales `dashboard.py.bak-*` (junio 10). Sin control de versiones: cuidado al editar, hacer copia antes de cambios grandes.
- Fuente de verdad del cableado (servicios, recuperación tras apagón, flujos): `docs/ARQUITECTURA-GENERACION.md` (en el repo). Esta memoria solo guarda lo no-obvio/cross-session.
- Tras editar `dashboard.py` o `velvet/forge.py`: `systemctl --user restart iagen-dashboard` (Python cachea `import forge`; sin restart sigue la versión vieja).

## Estado actual (2026-06-22): ya no es solo un generador, es pipeline completo de generación→publicación
El dashboard creció ~88% en líneas absorbiendo todo el post-proceso. Subsistemas añadidos (todos en `dashboard.py`, endpoints `/api/*`):
- **Captions multilingües** (`claude_caption` + `ollama_caption`): SIEMPRE 3 idiomas (ja/en/es), 1 línea por idioma; el usuario elige cuál al publicar. Más hashtags y alt-text.
- **Metadata Pixiv** (`build_pixiv_meta`): `{title, tags}` estilo Pixiv (japonés primero, sin `#`, máx 10) + `desc`=caption multilingüe sin hashtags.
- **Marca de agua** (`watermark_image`): branding `@KelsieDark` / Pixiv `kelsie_dark`, editable. Endpoint `/api/watermark` + `/api/watermark_text`.
- **Estado "publicado"** por imagen (`is_published`/`toggle_published`) y **rating embebido en el PNG** (`embedded_rating`/`set_image_rating`) con overrides.
- **Hires fix** (`run_hiresfix`, cola `hfix` + sugerencia de denoise/prompt vía Claude `hfix_suggest`). El hires es el **pico de presión de VRAM** del pipeline en la GPU de 4GB → pre-flight exige sd-server en `wai` e idle. `HIRES_BUDGET="1280x1920"` (2.46 MP, en prueba desde 2026-06-19, un escalón abajo del máximo).
- **Reroll / RELANZAR** (`run_reroll`, reusa el endpoint `/api/hires_run`): re-genera txt2img cada imagen marcada con su prompt/params embebidos y **seed nueva** (otra suerte de anatomía); quita score tags; sale a `<grupo>-reroll/`.
- **Inpaint** (`run_inpaint` + máscara) y **cutout** (`run_cutout` → rembg, recortar figura/Blob).
- **Sync UI** (`sync_list`/`run_sync`/`sync_status` + ignore list): antes el sync era solo `--sync` automático por imagen; ahora hay panel para traer carpetas desde Torre 1, ignorar y ver estado.
- **Gestión de galería en lote**: delete/move batch, limpieza de carpetas vacías, reboot Torre.
- **PWA**: thumbnails webp cacheados (`get_thumb`/`_cached_webp`), iconos y manifest → instalable como app.
- **Biblioteca de prompts** (`load_library`/`save_library`, `library.json`, endpoints `library_get`/`library_set`).

**Sigue siendo 3 modos** en `#gen-modal` (cajón/session/group), confirmado en el HTML (`data-mode`). `run_velvet_card` + `/api/velvet_card` existe pero NO es un 4º botón del modal (acción aparte por card).

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

## Generador unificado (modal, 2026-06-08) — implementado y validado
Los dos paneles (cajón + Velvet Forge en tabs) se unificaron en UN modal `#gen-modal` con
selector de 3 modos. Al ser modal, `genOpen()` entra en la guarda de `scheduleReload()` → ya
no se borran los tags al auto-refrescar. Modos:
- **Cajón** (`run_generate`, INTACTO): N personajes × seeds, cada uno solo. Salida `gen-<slug>`.
- **Sesión de personaje** (ex-Velvet Forge → `run_character_session` + `/api/character_session`,
  usa forge.py): 1 personaje × varias composiciones (filas) × N seeds. Salida `gen-<slug>`.
- **Escena grupal** (`run_group_scene` + `/api/group`): EXACTAMENTE 2 personajes en una
  composición × seeds. Salida `gen-<slug1>-<slug2>`. Tope 2 (sd.cpp sin regional prompting).
Los 3 con seeds (1-6, def 2) y steps (20-70, def 50). Spec/plan en docs/superpowers/ del proyecto.

## BUG "sin respuesta de Torre 1" — CAUSA RAÍZ: ARG_MAX (2026-06-14)
**Bloqueo real del usuario:** chains grandes (varios chars×seeds o composición larga) fallaban
SIEMPRE con "✗ sin respuesta de Torre 1". `b64_launch_remote` metía el script base64 en el ARGV de
ssh (`echo '<b64>' | base64 -d`); si el base64 pasa ~128KB (MAX_ARG_STRLEN), el subprocess revienta
al exec con `OSError [Errno 7] Argument list too long: 'ssh'`, ssh_run lo tragaba y devolvía "" ->
falso "sin respuesta de Torre 1". NO era Torre 1, ni SSH, ni Claude (el usuario sospechó de Claude;
una prueba real usó Claude OK). Scripts chicos no lo disparaban.
**Fix:** base64 POR STDIN (`subprocess.run(..., input=b)` + `base64 -d` remoto), no en argv. Sin
límite ARG_MAX. Verificado: script 207KB -> viejo=Errno7, nuevo=LAUNCHED 0.9s. Diagnóstico: se
instrumentó el mensaje rojo con rc/stderr/tiempo (`[rc=EXC 0s: Errno 7]`) -> rc=EXC+0s=ARG_MAX,
rc=255+5s=ConnectTimeout.

### (mismo fix) falso negativo por canal SSH colgado
Síntoma: al lanzar una chain, el dashboard mostraba "✗ sin respuesta de Torre 1" pero la chain
SÍ arrancaba en Torre 1. El usuario lo interpretaba como "Torre 1 caída". Al reintentar →
"Torre 1 ocupada" (porque la primera SÍ corría). El intento 2026-06-08 (`>/dev/null 2>&1` en
setsid) NO bastó: el `setsid ... & disown` seguía dejando el proceso de la chain reteniendo el
canal SSH → `ssh_run` llegaba a su timeout 25s → descartaba el stdout → out vacío → falso fallo.
Verificado midiendo: el lanzador con `& disown` hacía que el SSH retornara recién a los 25s.
**Fix real (b64_launch_remote):** patrón fire-and-forget con SUBSHELL — `( ( setsid bash -c
'nohup REMOTE >/tmp/N.out 2>&1' </dev/null >/dev/null 2>&1 & ) ; echo LAUNCHED )`. El `( cmd & )`
backgroundea y el subshell sale al instante → SSH retorna en ~1s con LAUNCHED y la chain corre.
`echo LAUNCHED` va dentro del `&&` de `bash -n` (sin falso positivo en error de sintaxis;
`SYNTAX_ERR` se reporta). Timeout bajado a 20s. b64_launch_remote es el ÚNICO punto de
lanzamiento (5 callers: cajón/sesión/grupal/hires/img2img) → arregla los 3 modos.
**Heurística para el usuario:** error rojo a ~5s = `ConnectTimeout=5`, SSH NO conectó (fallo real,
no lanzó, reintentar); error a ~25s = falso-negativo viejo (ya no debería ocurrir tras el fix).
Efecto secundario del bug viejo: cada lanzamiento colgado dejaba una conexión SSH abierta 25s y,
sumadas al loop del dashboard (~20 SSH/8s), saturaban por ratos el sshd de Torre 1 → parpadeos de
conexión de 5s. El fix también elimina esa acumulación.

## Mapa de prompts NEGATIVOS (dónde editarlos)
- Cajón + Grupal (`claude_build_prompt`): `dashboard.py:443` DEFAULT_NEG (principal); `:769`
  añade nsfw/nude si rating!=nsfw; `:702-711` PROMPT_SYS (instrucción al LLM); `:771` grupal
  quita 2girls/multiple girls.
- Sesión (forge.py): `forge.py:34` SAFETY_NEG="bad hands" (hardcoded); `velvet/velvet.json` →
  neg_safety(12), neg_base(13, principal), neg_solo_extra(14), bg_neg(20, fondos).
- gen.sh vive en Torre 1; el negativo se pasa como arg desde la chain.
- dashboard.py/forge.py → restart iagen-dashboard; velvet.json → sin restart.
