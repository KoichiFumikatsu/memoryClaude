# AI image generation — Estado actual (2026-05-30)

> **Doc operativo activo.** Para histórico de descartados ver `ai-image-local-history.md`.
> Doc largo con todos los detalles operativos en Torre 1: `~/projects/ia-gen/docs/setup-actual.md`.

## Hardware Torre 1 (post-upgrade 2026-05-29)

- GPU: AMD RX 570 **4GB VRAM** (Polaris, Vulkan/RADV, sin cambio)
- RAM: **15GB** (antes 8GB)
- CPU: **12 hilos**
- Backend único: **Vulkan** (ROCm/HIP descartado, crash en kernel 6.17)

## Stack actual — 4 modelos SDXL en `~/apps/sdcpp/models/checkpoints/`

| Modelo | Flag | Tamaño | Especialidad validada |
|---|---|---|---|
| `NoobAI-XL-v1.1.safetensors` | `noobai` | 6.7G | Likeness anime mainstream + NSFW Danbooru. Frieren biblioteca ⭐⭐⭐⭐⭐ |
| `animagine-xl-4.0-opt.safetensors` | `animagine` | 6.5G | Anime SFW puro refinado. Marine pirate ship ⭐⭐⭐⭐⭐ |
| `waiNSFWIllustrious_v14.safetensors` | `wai` | 6.5G | NSFW masivo + likeness. Zelda sleep NSFW ⭐⭐⭐⭐⭐ |
| `ponyRealism_v22MainVAE.safetensors` | `ponyrealism` | 6.7G | Semi-realismo, likeness anime POBRE. Mejor para originales |

## Mapa de uso por caso

| Caso | Modelo |
|---|---|
| SFW + likeness anime + escenario detallado | **`wai`** o `noobai` (empatados) |
| SFW estilo manga saturado / colores vivos | `animagine` |
| **Manga PÁGINA real multi-panel + speech bubbles** | **`noobai`** (interpreta literal) |
| **Manga ilustración single panel pulida** | **`wai`** |
| NSFW masivo (sleep_sex, fellatio, harassment) | **`wai`** líder |
| **NSFW + manga B&N simultáneo** | **`wai`** (validado Yvette sleep manga) |
| **Batches NSFW largos multi-personaje** | **`wai`** (validado 20 imgs Hololive) |
| Cellshading 3D Genshin/HSR/BOTW | **pendiente** (chain Xinyan corriendo al cierre 2026-05-31) |
| Semi-realismo / personajes originales | `ponyrealism` |
| EVITAR para personajes anime específicos | `ponyrealism` |

**Hallazgos clave 2026-05-30/31:**
- WAI cubre TODO: SFW + NSFW + manga + likeness. El más completo del stack.
- NoobAI interpreta `manga style` literal → genera **PÁGINA real con viñetas multi-panel + speech bubbles + screentones**. Único para esto.
- Animagine se queda en single panel saturado / cartoon. Útil para estilo "manga colorido".
- Pony Realism descartado para personajes anime específicos (likeness pobre).

## Cambio de modelo

```bash
~/projects/ia-gen/scripts/switch-model.sh {noobai|animagine|wai|ponyrealism|<ruta>}
```

## Prompt format por familia

**`wai`, `ponyrealism` (Pony forks)** — score tags OBLIGATORIOS:
```
POS: score_9, score_8_up, score_7_up, source_anime, rating_explicit, (masterpiece:1.2), ...
NEG: score_1, score_2, score_3, source_furry, source_pony, (worst quality:1.4), ...
```

**`noobai`, `animagine` (Illustrious-style)** — Danbooru puro SIN score tags:
```
POS: (masterpiece:1.2), [danbooru tags], anime style, 2d
NEG: (worst quality:1.4), deformed, ..., realistic, western style
```
Tags multipalabra con underscore: `cum_in_mouth`, `sleep_sex`, `looking_at_viewer`.

## Settings probados

- Resolución: `832x1216` (vertical SDXL). NO 1024×1024 (OOM VAE).
- Steps: 50 (sweet spot)
- CFG: 6.5
- Sampler: `dpm++2mv2`, Scheduler: `Karras`, Clip skip: 2
- Tiempo: ~22s/step → **~20 min por imagen 50 steps**
- Switch entre modelos: ~30s

## Stack auxiliar Torre 1

- **Binary:** `/home/kelsielinux/apps/sdcpp/build/bin/sd-server`
- **Launch:** `~/apps/sdcpp/launch.sh` (acepta ruta a modelo como arg)
- **API:** `http://localhost:7860` (compatible WebUI A1111, solo desde Torre 1)
- **sdcpp.service:** **DISABLED** desde 2026-05-29 (`Restart=on-failure` causaba duplicación → saturación VRAM). Arrancar manual.
- **controlnet-aux** en venv: preprocessor foto→OpenPose esqueleto funcional. Reutilizable si se setupea ControlNet en otro entorno (sd.cpp SDXL = zona muerta confirmada).
- **CivitAI token** en `~/.config/civitai_token` (Torre 1)
- **LoRAs descargadas** en `~/apps/sdcpp/models/loras/` (perfect_hands_v2, sdxl_detail, smooth_anime_style_xl) — **NO integradas**: `merge_lora.py` con bug de namespace kohya↔SDXL pendiente.

## Scripts en `~/projects/ia-gen/scripts/`

| Script | Función |
|---|---|
| `gen.sh` | txt2img endurecido (exit codes 2-7, timeout, WARN <50KB) |
| `img2img.sh` | img2img mismos flags + `--denoising` + `--sync` |
| `switch-model.sh` | aliases noobai/animagine/wai/ponyrealism |
| `upscale.sh` | waifu2x 2x |
| `sync-to-fumilinux.sh` | rsync outputs → `kelsie@100.116.50.47:~/Pictures/ia-gen/` |
| `merge_lora.py` | **BUG pendiente** (namespace) |

### gen.sh autodetect

Lee modelo del proceso sd-server y normaliza filename:
- `NoobAI*` → `noobai`
- `animagine*` → `animagine`
- `waiNSFW*` → `wai`
- `*Realism*` → `ponyrealism`
- otros → `model`

Output: `outputs/CHAR/{modelo}_seed{N}_{timestamp}_{i}.png`

## Workflow probado

```bash
ssh kelsielinux@100.67.216.43
cd ~/projects/ia-gen/scripts
./switch-model.sh noobai
./gen.sh "POS" "NEG" --size 832x1216 --steps 50 --cfg 6.5 --sampler dpm++2mv2 \
  --seed N --char personaje --sync
```

Las imágenes aparecen en `~/Pictures/ia-gen/CHAR/` en Fumilinux por --sync automático.

## Tests de validación realizados (2026-05-30)

### Individuales
| Test | Modelo | Likeness | Composición |
|---|---|---|---|
| Marine pirata SFW | animagine | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Zelda montaña SFW | animagine | ⭐⭐ (lejana) | ⭐⭐⭐⭐ |
| Marine sleep NSFW | wai | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Zelda sleep NSFW | wai | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Frieren biblioteca SFW | noobai | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Power CSM SFW | ponyrealism | ⭐ | ⭐⭐⭐⭐ |

### Chain comparativo Marine kitchen SFW (seed 7777, 3 modelos)
| Modelo | Likeness | Composición | Detalle escenario |
|---|---|---|---|
| `noobai` | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| `animagine` | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| `wai` | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| `ponyrealism` | (cancelado por usuario, descartado para anime) |

### Chain manga humor SFW (Marine viñeta, seed 9999, 3 modelos)
| Modelo | Interpretación | Estilo manga |
|---|---|---|
| `noobai` | **PÁGINA REAL multi-panel (3 viñetas)** | ⭐⭐⭐⭐⭐ |
| `animagine` | Single panel grande | ⭐⭐⭐⭐ |
| `wai` | Single panel ilustración pulida | ⭐⭐⭐⭐⭐ |

### Yvette sleep manga NSFW (seed 5454, solo WAI)
WAI demostró **3 capacidades simultáneas en una imagen:** likeness Yvette en B&N + composición NSFW (sleep+oral+cum) + estilo manga single panel.

### Batch nocturno Hololive NSFW (20 imgs, solo WAI, 2026-05-30)
4 personajes (Calliope, Gura, Kiara, Ina) × 5 composiciones (2 sleep + 2 blowjob + 1 manga bukkake) = 20 imgs generadas en ~6.7h sin alertas. Validado para batches largos.

### Chain Xinyan cellshading 3D Genshin (pendiente al cierre 2026-05-31)
SFW Xinyan+HuTao concert Liyue + NSFW Xinyan sleep cum × 3 modelos = 6 imgs. Tags clave: `(cel shading:1.4), (gradient cel shading:1.2), (anime 3d:1.2), (genshin impact style:1.3)`. Negative: `flat color anime, traditional 2d anime, hand-drawn`. Outputs en `~/Pictures/ia-gen/xinyan-genshin-compare/`.
