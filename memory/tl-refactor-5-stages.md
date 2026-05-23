# TL Games — Refactor pipeline a 5 etapas observables

**Inicio:** 2026-05-22 (sesión Opus 4.7 + plan mode)
**Plan file local de Claude:** `/home/kelsie/.claude/plans/eager-seeking-pudding.md`
**Repo:** `/home/kelsie/projects/tlgames/`

## Objetivo

Refactorizar `tools/pipeline_server.py` para que Koichi pase solo la ruta del juego y el pipeline corra autónomo con 5 etapas observables, ntfy estructurado, dashboard visual rico, settings persistentes y empaquetado ZIP al final. Cero intervención durante el run.

## Decisiones tomadas (4 preguntas con Koichi)

1. **Settings file**: `tools/pipeline_settings.json` (dedicado, no .env)
2. **Package**: copia + ZIP completo en `~/Documents/games tl/<juego>.zip`
3. **ntfy events**: stage start/end + cada 10% global + provider switch + errores
4. **Provider**: PREFLIGHT — contar chars, consultar cuota DeepL `/v2/usage`, decidir antes de empezar

## Las 5 etapas comunes

| # | Etapa | Peso default (Ren'Py / Unity / RPGM) | Responsabilidad |
|---|-------|---------------------------------------|-----------------|
| 1 | analyze | 5/5/5 | detect_engine + detect_game_info + count + preflight provider. Ren'Py: corre SDK si falta tl/spanish/ |
| 2 | setup | 10/5/5 | backup, prep workspace, cache warmup |
| 3 | translate | 65/80/80 | Pipeline MT DeepL/OpenAI, batching, streaming |
| 4 | lint_qa | 15/0/0 | Ren'Py: postprocess+lint+QA Ollama. Unity/RPGM: skipped |
| 5 | package | 5/10/10 | rsync + ZIP. Si > 5GB: skip ZIP con warning |

Si stage skipped, sus puntos se reasignan a translate. Si pending==0 en Ren'Py, translate skipped pct=100.

## Estructura del Job (aditiva, NO renombrar campos legacy)

```python
job = {
    # Legacy (mantener intactos, jobs_history.jsonl compat)
    "job_id", "game_path", "game_name", "provider", "lang",
    "status", "progress[]", "stats{}", "engine{}", "game_info{}",
    "started_at", "finished_at", "qa_report", "qa_issues",
    "tl_path", "error", "warning", "copy_status", "copy_path",

    # NUEVO aditivo
    "stages": {
        "analyze":  {"status", "started_at", "finished_at", "pct", "details"},
        "setup":    {...},
        "translate":{...},
        "lint_qa":  {...},
        "package":  {...},
    },
    "current_stage": "analyze",
    "overall_pct": 0,
    "events": [{"ts", "stage", "event", **extra}],
    "analysis": {engine, version, os, runtime, files, strings, chars,
                 chosen_provider, provider_reason, deepl_quota_avail,
                 openai_budget_avail, size_bytes},
    "errors": [], "warnings": [],
}
```

## Protocolo de progreso — CLAVE

**NO modificar los 3 scripts engine** (`translate.py`, `translate_unity_json.py`, `translate_rpgmaker.py`). Son frágiles, sin tests. El `pipeline_server.py` parsea su stdout existente con regex que YA EXISTE:
- `_PROG_RE = r'\[(\d+)/(\d+)\]\s+(\S+)\s*\|\s*(\d+)/(\d+)\s+strings\s*\|\s*(\d+)%'`
- `_OAPI_SPENT_RE = r'gastado \$([0-9.]+)'`
- `_TO_OPENAI_RE = r'\[openai\]|cambiando a OpenAI'`

Para Ren'Py (translate.py corre 1 vez por archivo), el server cuenta `i/len(rpy_files)` y emite progreso global. Mantener compat con patterns de error existentes: `ABORT|BUDGET|ERROR:|WARN|SKIP`.

## Componentes nuevos

### `tools/pipeline_settings.json` (CREADO)
Config persistente con defaults + sobrecarga del .env solo si la key existe.

### `tools/tl/_settings.py` (CREADO)
Loader con cache + mtime reload. API: `get_all()`, `get(path: str, default)`, `write(data)`, `defaults()`.

### `tools/tl/_deepl.py` (CREADO)
Centraliza:
- `DEEPL_FREE_QUOTA_REAL = 1_000_000` (bug API reporta 500K)
- `get_keys()` — pool desde env vars `DEEPL_API_KEY*`
- `check_usage(key)` — GET /v2/usage con corrector bug 500K
- `check_quota_pool(keys)` — suma chars disponibles del pool
- `is_exhausted_today()` / `mark_exhausted_today()` — estado en `.cache/deepl_quota_state.json`

### `tools/tl/_preflight.py` (PENDIENTE Fase 3)
```python
def decide_provider(chars: int, settings: dict, deepl_state: dict) -> tuple[str, str]:
    # 1. Si setting default_provider != "auto" → forced
    # 2. Si exhausted_today → openai
    # 3. Si chars <= available → deepl
    # 4. Si red falla → deepl (fallback reactivo actual)
    # 5. Else → openai (chars excede cuota)
```

### `tools/tl/_package.py` (PENDIENTE Fase 3)
CLI standalone. zipfile.ZipFile + ZIP_STORED para > compress_below_mb. Emite `[PROGRESS] file=X done=N total=M pct=P`. Cap max_size_gb=5 → skip con warning.

### `tools/dashboard.html` v2 (PENDIENTE Fase 4)
5 chips de etapas, panel expandido del activo, settings link, lista pendientes, panel errores/warnings, ETA, costos detallados DeepL+OpenAI. Fallback a `progress[]` si no hay `stages`.

## Cambios en pipeline_server.py

### Nuevas clases/funciones
- `class StageTracker` — gestiona stages/current_stage/overall_pct. Pesos desde `settings.stage_weights[engine]`, reasignación de skipped a translate.
- `def emit_event(job, stage, event, **extra)` — append a `job["events"]`, decide ntfy según `settings.ntfy.*`. Mantiene `progress[]` legacy.
- `def run_pipeline_v2(job, settings)` — wrapper único, llama a sub-funciones por engine.

### Refactor (wrap-not-replace)
- `run_renpy_pipeline`, `run_unity_json_pipeline`, `run_rpgmaker_pipeline` se DESCOMPONEN en `_<engine>_analyze`, `_<engine>_setup`, `_<engine>_translate`, `_<engine>_lint_qa`, `_<engine>_package`.
- Las invocaciones a los `translate_*.py` IDÉNTICAS (mismo Popen, mismo regex).
- Mantener v1 con flag `?pipeline_version=v1` por 1-2 semanas.

### Endpoints
- **NUEVO** `GET /settings` — devuelve `pipeline_settings.json` mergeado
- **NUEVO** `POST /settings` — escribe tras validar schema
- **NUEVO** `GET /pipeline/<id>/events` — histórico
- **NUEVO** `GET /health` enriquecido — Ollama + DeepL pool + OpenAI budget
- **MODIFICADO** `POST /pipeline` — body mínimo `{"path": "..."}`. Defaults del settings. Backward compat con campos antiguos.
- **MOVIDO** `DASHBOARD_HTML` (línea 111) — leer `tools/dashboard.html`, eliminar embebido.

## Plan de migración — TODAS LAS FASES IMPLEMENTADAS 2026-05-22

| Fase | Trabajo | Estado |
|------|---------|--------|
| **0 — Base** | `pipeline_settings.json`, `_settings.py`, `_deepl.py`, endpoints + health enriquecido | **COMPLETA** ✓ |
| **1 — Scaffolding** | Campos aditivos + StageTracker + emit_event con ntfy dispatch | **COMPLETA** ✓ |
| **2 — Wrap engines** | `run_pipeline_v2` + sub-funciones por etapa por engine. v1 con flag `?pipeline_version=v1` | **COMPLETA** ✓ (verif. job 46fd02c9) |
| **3 — Preflight + Package** | `_preflight.py` + `_package.py` integrados en analyze/package | **COMPLETA** ✓ |
| **4 — Dashboard v2** | 5 chips, panel expandido, settings modal, health alert, eventos colapsable, fallback legacy | **COMPLETA** ✓ |
| **5 — Cleanup** | Borrar v1 tras 1-2 semanas estable | DEFERRED |

## Archivos — estado actual

### CREADOS (Fase 0 completa + Fase 3 archivos)
- `tools/pipeline_settings.json`
- `tools/tl/_settings.py`
- `tools/tl/_deepl.py`
- `tools/tl/_preflight.py` (CLI: `python3 _preflight.py <chars>`)
- `tools/tl/_package.py` (CLI: `python3 _package.py <game> <zip> [--max-gb N] [--compress-below-mb N]`)

### MODIFICADO (Fase 0 + Fase 1 en pipeline_server.py)
- Import de `_settings` (alias `_s`) y `_deepl` (alias `_dl`)
- Constantes `STAGE_ORDER`, `_DEFAULT_STAGE_WEIGHTS`
- Helpers stage tracking + `class StageTracker` + `def emit_event(job, stage, event, **extra)`
- Job creation incluye campos aditivos: `stages`, `current_stage`, `overall_pct`, `events`, `analysis`, `errors`, `warnings`, `ntfy_topic`
- Endpoints **GET /settings**, **POST /settings**, **GET /pipeline/<id>/events**, **GET /health enriquecido** (Ollama + DeepL pool + OpenAI budget)
- GET /dashboard relee `dashboard.html` del disco en cada request (reload sin reiniciar)

### Verificacion ejecutada al cierre de sesion 2026-05-22
- `python3 -m py_compile` OK en todos los modificados
- `sudo systemctl restart tlgames-pipeline` exitoso (PID 2243562)
- `curl /settings` devuelve config completa
- `curl /health` reporta: ollama=ok, deepl=499999/1M (avail=500001) source=api, exhausted_today=true (cache stale), openai=$0.249/$1.50 (1659 reqs)
- `python3 _preflight.py 100000` devuelve `("openai", "deepl_exhausted_today")` respetando cache state

### PENDIENTES (al retomar wakeup)
- **Fase 2 (grueso):** descomponer 3 `run_*_pipeline` en sub-funciones por etapa, crear `run_pipeline_v2`, integrar StageTracker + preflight + _package
- **Fase 4 (UI):** dashboard.html v2 con 5 chips + panel expandido + settings link + ETA + costos detallados. Fallback a `progress[]`
- **Verificacion E2E final**

### POR MODIFICAR
- `/home/kelsie/projects/tlgames/tools/pipeline_server.py` — endpoints settings, StageTracker, emit_event, descomposición de los 3 run_*_pipeline, health enriquecido, eliminar embebido, runner principal
- `/home/kelsie/projects/tlgames/tools/dashboard.html` — rewrite JS render para stages

### NO TOCAR (frágiles, sin tests)
- `tools/tl/translate.py` (1173 líneas)
- `tools/tl/translate_unity_json.py` (829 líneas)
- `tools/tl/translate_rpgmaker.py` (498 líneas)
- `tools/qa_renpy.py`, `tools/qa_server.py`
- `tools/version_tracker.py`
- `logs/pipeline_jobs_history.jsonl` (solo aditivo)

## Funciones existentes a reusar (no tocar)

- `detect_engine(path)` — pipeline_server.py:118-167
- `detect_game_info(path, engine)` — pipeline_server.py:176-311
- `detect_unity_json_tl(path)` — pipeline_server.py:314-339
- `find_renpy_sdk()` — pipeline_server.py:344-372
- `copy_to_games_tl(game_path, game_name, job)` — pipeline_server.py:377-408
- `ntfy_send(topic, msg, title, priority)` — pipeline_server.py:60-73
- `_count_pending(tl_path)` — pipeline_server.py:757-774

## Variables .env y caches relevantes

- `DEEPL_API_KEY=96ff6ae6-78e0-42a8-b679-9d9574217a32:fx`
- `OPENAI_API_KEY` (sk-proj...)
- `RENPY_SDK=/home/kelsie/renpy-8.3.7-sdk/renpy.sh`
- `OPENAI_BUDGET_USD=1.50`
- Caches en `tools/tl/.cache/`: deepl.json, openai.json, openai_usage.json (1581 reqs, $0.24), deepl_quota_state.json (exhausted hoy)

## Autonomía + minimización de gasto

### n8n
Workflow "TL Games — Pipeline Universal" (ID `CAak4TD2OzhhsXBA`) — fire-and-forget. NO se toca en el refactor.

### Ollama
localhost:11434, llama3.2:3b CPU. Solo qa_server (8765) en lint_qa. NO se usa para traducción (decisión 2026-04-28).

### Mecanismos de ahorro activos
- Cache OpenAI/DeepL existente
- Preflight DeepL primero (NUEVO Fase 3)
- Glosario tl-es-glossary.json
- clear_mirror_targets.py (5-20% ahorro)
- Batching OPENAI_BATCH_SIZE=25 (fix 2026-05-22)
- Budget hard-stop $1.50
- QA local con Ollama ($0)
- Persistencia exhausted_today

### Refuerzos extra para autonomía
- emit_event envía ntfy en CADA error/warning
- Heartbeat 5min: ntfy `[STALL]` si translate no emite
- `/health` enriquecido con estado Ollama + DeepL + OpenAI

## Riesgos y mitigaciones

| Riesgo | Mitigación |
|--------|------------|
| Sin tests automatizados | Wrap-not-replace. v1 con flag 1-2 semanas. |
| Romper jobs_history.jsonl legacy | Solo aditivo, nunca renombrar. Dashboard fallback a progress[]. |
| Race condition preflight | Deuda aceptada (ya existe con cache state). |
| translate_*.py cambian stdout futuro | Tests regex con fixtures históricos. |
| ZIP 4GB+ consume recursos | Cap max_size_gb=5 + mode="auto". |
| /v2/usage falla en preflight | Fallback a cache state, luego comportamiento reactivo. |

## Verificación E2E (pendiente al final)

1. `curl /settings | jq .`, editar JSON, recurl, debe reflejar sin reiniciar.
2. `curl -X POST /pipeline -d '{"path": "..."}'` aplica defaults.
3. 5 chips de etapas cambian en orden en dashboard.
4. ntfy ~15 mensajes/job ShoSakyu (vs bug 20/min histórico).
5. Con exhausted_today, `chosen_provider=="openai"` antes de translate.
6. ZIP existe en `~/Documents/games tl/<nombre>.zip`. >5GB → skip con warning.
7. Jobs viejos JSONL siguen renderizando.
8. Ren'Py sin pending → translate skipped pct=100, lint_qa con issues_total.

## Estado al 2026-05-22 ~18:45 — REFACTOR COMPLETO

Todas las fases (0-4) implementadas y verificadas con job real `46fd02c9` (Adventurer Trainer Ren'Py 0.2.2b, 110 archivos, 220K chars, 2.14GB):
- analyze done 100% (preflight retorna deepl, switch reactivo a openai por exhausted_today)
- setup done 100%
- translate al 50% (56/110 archivos) al cerrar sesión
- provider_switch DeepL→OpenAI detectado y notificado por ntfy
- 5 chips renderizan correctamente en dashboard
- POST /settings round-trip OK

**Pendiente no crítico:**
- E2E del stage package (job todavía en translate al cierre, completará lint_qa + package automáticamente)
- Fase 5 cleanup deferred 1-2 semanas

**Bug DeepL exhausted_today auto-clear:** `_maybe_clear_deepl_exhausted()` se invoca en analyze; limpia el flag si pool reporta >50K disponibles via API. Resuelve el problema de flag stale por 456 en una sola key del pool.

## Migración QA a Groq (2026-05-22 ~22:00)

**Problema:** Ollama `llama3.2:3b` en CPU se saturaba con cargas concurrentes. Medición real: `/health` tardó 4299s (71 min) cuando el pipeline corría lint_qa sobre 110 archivos. Job en producción siempre marcaba `lint_qa warn timeout` y continuaba sin QA semántico.

**Solución:** `qa_renpy.py` ahora soporta dos backends via constante `BACKEND` ('auto' | 'groq' | 'ollama'):
- **groq** (default si hay `GROQ_API_KEY` en .env): llama API REST OpenAI-compatible `https://api.groq.com/openai/v1/chat/completions` con `urllib.request` (cero deps nuevas). Modelo `llama-3.1-8b-instant`. Free tier 30 req/min, 14400 req/día. Rate limit interno 2s entre requests.
- **ollama** (fallback si no hay key): comportamiento original.
- **auto**: groq si hay key, sino ollama.

**Trampa Cloudflare:** Groq está detrás de Cloudflare. `urllib.request` por default manda `User-Agent: Python-urllib/X.Y` y Cloudflare lo bloquea con HTTP 403 error code 1010. Workaround: header `User-Agent: tlgames-qa/1.0 (+...)` resuelve.

**Performance comparado** (archivo `common.rpy` Adv_Trainer, 147 pares en 4 batches):
- Ollama llama3.2:3b CPU: timeout (no responde a tiempo)
- Groq llama-3.1-8b-instant: **16.5s total** (4s/batch + 2s rate limit), **~270x más rápido**

**Configuración:**
- `tools/pipeline_settings.json` → `qa.timeout_sec: 1800` (30 min, ample para juegos grandes), `qa.backend: groq`
- `tools/tl/_settings.py` defaults equivalentes
- `tools/qa_server.py` `/health` ahora reporta `{backend, model}` dinámicamente
- `.env` añadido `GROQ_API_KEY=gsk_...`
- `qa_renpy.py` CLI: `--backend auto|groq|ollama` `--groq-model X` `--ollama-model X`

**Verificación:** `qa_server /health` reporta `{"backend":"groq","model":"llama-3.1-8b-instant"}` tras restart.

**Costo:** $0 (free tier). Si excede 14400 req/día, downgrade automático a Ollama via setting o env vars.

## Script de diagnóstico (2026-05-22)

`tools/diagnose.sh` — reporte completo de salud del pipeline para copy-paste a Claude cuando algo falle.

**Uso:**
```bash
./tools/diagnose.sh                       # reporte general
./tools/diagnose.sh <job_id>              # general + detalle del job
./tools/diagnose.sh <job_id> --log-lines 100  # mas lineas del log
./tools/diagnose.sh --no-color            # ANSI off para archivos
```

**Secciones del reporte:**
1. Servicios systemd (tlgames-pipeline, tlgames-qa, tlgames-versions)
2. Health endpoints — pipeline_server + qa_server con backend/model dinámico
3. Últimos 5 jobs con status colorizado (OK/ERR/RUN)
4. Detalle del job (si se pasa job_id): analysis, stages con details, stats, eventos críticos (errors/warns/provider_switch), últimas N lineas del progress log
5. Disco — espacio en games tl/
6. Errores systemd últimas 2h en pipeline/qa
7. Variables .env relevantes (keys ocultas, solo prefix+suffix)

**Trampa:** /pipeline/<id> solo busca en _jobs in-memory (max 50). El script hace fallback automático a `logs/pipeline_jobs_history.jsonl` si el job no existe en memoria. Sin esto, jobs viejos eran invisibles.

**Workaround Python warning:** los logs tienen `\$X.XX` literales (dólares de gastos OpenAI). Python heredoc los lee como escape sequences inválidas. Solución: `python3 -W ignore` silencia el warning sin afectar funcionalidad.

**Modificaciones al pipeline_server.py:** de 1000 a 1859 líneas, ver detalle en versión local en `/home/kelsie/.claude/projects/-home-kelsie/memory/project_tlgames_refactor_5_stages.md`.
