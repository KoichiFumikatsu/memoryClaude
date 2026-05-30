# AI image generation — Histórico de descartados (NO reintroducir)

> Para estado actual ver `ai-image-local.md`. Este archivo documenta lo que **NO debe usarse**.
> Doc largo del histórico en Torre 1: `~/projects/ia-gen/docs/setup-descartado.md`.

## Modelos eliminados — NO descargar de nuevo

### Borrados 2026-05-30 (reemplazados por los 4 forks)
- **`PonyDiffusionV6XL.safetensors`** (6.5G) — Pony V6 vanilla. Likeness pobre para personajes anime mainstream. Anatomía variable. Reemplazado por `wai` y `ponyrealism`.
- **`Illustrious-XL-v1.0.safetensors`** (6.5G) — Illustrious vanilla. NSFW limitado. Reemplazado por `noobai`.

### Borrados 2026-05-29 (no funcionaban o sin uso)
- **`PonyDiffusionV6XL-Q5_0.gguf`** (2.9G) — GGUF Q5_0 sobre SDXL **genera ruido puro** en sd.cpp. Nunca usar GGUFs de SDXL, solo safetensors fp16.
- **`MeinaHentai-baked-VAE-Q5_0.gguf`** (1.6G) — mismo problema GGUF.
- **`MeinaHentai-baked-VAE.safetensors`** (2G, SD 1.5) — fallback nunca usado en práctica.

### Embedding TÓXICO borrado 2026-05-29
- **`boring_sdxl_v1.safetensors`** (33KB) — Con Pony en SDXL + prompt complejo (40+ tags) + 50-70 steps producía **PNG blanco determinista de exactamente 35,507 bytes** (no ruido, mismo tamaño cada intento). Quitarlo del negative → imagen real de 2 MB. **NO descargar ni referenciar** en negative de SDXL.

## ControlNet en sd.cpp SDXL — zona muerta confirmada

### Causa raíz
`~/apps/sdcpp/src/control.hpp` solo carga ControlNets con tensors `input_blocks.X.X` y `zero_convs.X.X` (formato vanilla original lllyasviel). **TODOS los OpenPose SDXL públicos en HuggingFace están en formato diffusers** (`controlnet_down_blocks.X`, `controlnet_mid_block`) porque se entrenaron con la librería diffusers.

### Verificados incompatibles
- `xinsir/controlnet-openpose-sdxl-1.0` (2.4G) — error `zero_convs.5.0.bias not in model file`
- `thibaud/controlnet-openpose-sdxl-1.0` (5G) — formato diffusers
- `lllyasviel/sd_control_collection` (LLLite y T2I Adapter no soportados por sd.cpp)

### Caminos posibles si se retoma
1. Converter diffusers→vanilla (3-5h coding, mapping 844 tensors)
2. Cloud (RunPod/Vast.ai) con ComfyUI/A1111
3. Live USB / dual-boot kernel 5.x con ROCm funcional en Polaris
4. Upgrade GPU (RTX 3060 12GB ~$200 usada)

### Lo único reutilizable del intento
- **`controlnet-aux`** instalado en `~/projects/ia-gen/.venv/` (Torre 1) — preprocessor foto→OpenPose esqueleto. Funciona. Reutilizable si se setupea ControlNet desde otro entorno.

## Hardware obsoleto

### Torre 1 pre-upgrade — 8GB RAM
Reemplazado 2026-05-29. Causaba memory pressure: sd-server (6.8GB) + sistema → Pony generaba PNGs blancos en producción aunque test ligero pasaba. Test ligero (5 steps, 512×768) **NO es indicativo** — siempre probar a resolución y steps reales antes de un batch.

## Bugs históricos arreglados

### `gen.sh` / `img2img.sh` originales (pre-2026-05-29)
- `curl` sin `--max-time` → si sd-server colgaba, gen.sh esperaba infinito.
- Sin chequeo HTTP status ni JSON valid → response inválido = python fallaba silencioso, gen.sh reportaba "Sync OK" y `ls -t outputs/CHAR/{model}_seed{N}_*.png | head -1` agarraba imagen vieja con falso éxito.
- `pgrep -a sd-server | head -1` con >1 sd-server agarraba uno al azar (estado ambiguo).
- Autodetect modelo solo conocía "Pony" e "Illustrious" — modelos nuevos caían en `model`.
- img2img.sh sin `--sync` (inconsistencia).

**Fix aplicado 2026-05-29:** exit codes 2-7, timeout `steps × 60 + 300` seg, retry 1, WARN si PNG <50KB, abort visible si 0 o >1 sd-servers, autodetect con case de los 4 nuevos modelos.

### `sdcpp.service` con `Restart=on-failure`
**Disabled 2026-05-29.** Causaba duplicación: al matar sd-server con `pkill`, systemd lo relanzaba 5s después. Si yo lanzaba uno manual también, terminaban 2 procesos saturando VRAM → requests colgados silenciosos. Service ahora arranca solo manualmente.

### `merge_lora.py` namespace kohya↔SDXL (PENDIENTE)
Las LoRAs (perfect_hands_v2, sdxl_detail, smooth_anime_style_xl) están en formato kohya-ss (`lora_te1_*`, `lora_unet_*`). Pony base usa formato SDXL (`conditioner.embedders.0.*`, `model.diffusion_model.*`). merge_lora.py actual no traduce → reporta "0 deltas aplicados" → checkpoint es copia bit-a-bit del base. **Bug pendiente de fix**, las LoRAs siguen descargadas pero no integrables.

### Bash bug del `pgrep -c X || echo 0`
Produce "0\n0" cuando X falla → rompe checks numéricos. Usar `pgrep X | wc -l` que siempre devuelve un número.

## Packs OpenPose descargados y borrados 2026-05-30

Sin ControlNet funcional en sd.cpp eran inútiles:
- `openposeNSFWPosePackage` (67M, 525 poses NSFW)
- `thousandsOfPosesFrom_v10` (1.9G, 14,384 archivos)
- `actionPoses_climbingV20` (19M)

Re-bajables vía CivitAI API si se retoma ControlNet desde otro entorno.

## Lecciones documentadas (NO repetir)

1. **No confiar en test ligero** (5 steps, 512×768) — pueden pasar mientras producción (50 steps, 832×1216) falla silenciosa. Test real con specs de producción siempre.
2. **NO `boring_sdxl_v1`** en negative SDXL.
3. **NO GGUFs Q5_0** para modelos SDXL en sd.cpp — solo safetensors fp16.
4. **ControlNet OpenPose XL en sd.cpp = zona muerta**. Si lo necesitas, cloud o cambio de infra.
5. **`pgrep -c X || echo 0`** está roto, usar `pgrep X | wc -l`.
6. **`ls -t | head -1`** sin chequeo de exit code agarra imagen vieja si la gen falló — exit codes en gen.sh previenen esto ahora.
