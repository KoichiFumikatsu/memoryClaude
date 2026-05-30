# AI image generation — Estado y aprendizajes

**Workspace de proyecto:** `~/projects/ia-gen/` — README + symlinks a `outputs/` y `models/` + `scripts/` con wrappers + `prompts/` + `video/` (placeholder futuro). Ver `~/projects/ia-gen/README.md` y `~/projects/ia-gen/CLAUDE.md` (este último se autocarga al abrir Claude Code en el workspace, contiene config funcional default).

**Launcher Desktop:** `~/Desktop/ia-gen-claude.desktop` — abre terminal en `~/projects/ia-gen/` y arranca `claude`. Junto a `sdcpp-webui.desktop` (port 7860) y `open-webui.desktop` (port 3000).

**Estado actual Torre 1 (2026-05-27):** `stable-diffusion.cpp` compilado con backend Vulkan (RADV). ROCm/HIP descartado — crash irrecuperable en kernel 6.17. Stack activo: Vulkan + PonyDiffusion V6 XL safetensors fp16.

**Hardware GPU:** AMD Radeon **RX 570** (Ellesmere/Polaris 10), **4GB VRAM**. Confirmado por `vulkaninfo` + `mem_info_vram_total = 4294967296 bytes`. Esto explica el techo de 832×1216 (con VAE decode no entra 1024×1024). Pony fp16 (6.5GB) corre por swapping transparente que hace sd.cpp con `--fa --vae-tiling`. **VRAM 4GB sigue siendo el cuello para entrenamiento LoRA local.** Upgrade pendiente sería RTX 3060 12GB.

**Hardware CPU/RAM (upgrade 2026-05-29):** Torre 1 ahora tiene **15GB RAM (antes 8) + 12 hilos CPU**. El upgrade resolvió un bug crítico donde Pony generaba PNGs blancos de 34KB a 832×1216 + 20-70 steps (memory pressure causaba fallo silencioso en VAE decode). Post-upgrade Pony y Illustrious funcionan correctamente. LoRAs runtime sigue limitado por VRAM 4GB; pre-merge con `merge_lora.py` ahora viable.

**Plan a futuro:** seguir pruebas de imagen → empezar generación de video (evaluar AnimateDiff, SVD, CogVideoX, LTX-Video).

**Ruta actual:** `stable-diffusion.cpp` + Vulkan/RADV + PonyDiffusion V6 XL (SDXL). No usa ROCm ni HIP. GPU detectada como `RADV POLARIS10`, Vulkan 1.4.318.

## Open WebUI + Ollama (apoyo para prompts)

- Container Docker `open-webui` en `localhost:3000`, auto-restart, volumen `open-webui`.
- Pre-configurado con 3 providers OpenAI-compatibles via env vars al `docker run`:
  - OpenAI (`https://api.openai.com/v1`)
  - Groq (`https://api.groq.com/openai/v1`)
  - Gemini (`https://generativelanguage.googleapis.com/v1beta/openai`)
- Keys cargadas desde `~/projects/tlgames/.env` (OPENAI_API_KEY, GROQ_API_KEY, GEMINI_API_KEY).
- Ollama reconfigurado con override systemd `OLLAMA_HOST=0.0.0.0:11434` (default era 127.0.0.1, container no llegaba).
- Icono Desktop: `~/Desktop/open-webui.desktop`.
- **Limitación RAM:** Torre 1 solo tiene 8GB RAM, no 24GB como Fumilinux. llama3.2:3b (2.3GB) no carga si sd-server + Open WebUI + Firefox están activos simultáneamente. Para generación de prompts via Ollama desde script: descargar `llama3.2:1b` (1.3GB) o liberar RAM.
- Script `gen-from-desc.sh` en `~/apps/sdcpp/` traduce descripción ES → prompt Pony via Ollama → genera. Funcional pero limitado por RAM disponible.

## Stack actual Torre 1

- **Binary:** `/home/kelsielinux/apps/sdcpp/build/bin/sd-server` y `sd-cli`
- **Launch:** `/home/kelsielinux/apps/sdcpp/launch.sh` — arranca servidor en `0.0.0.0:7860`, backend `vulkan0`, flags: `--fa --vae-tiling`
- **Generación rápida:** `/home/kelsielinux/apps/sdcpp/gen.sh "PROMPT" "NEGATIVE" [flags]`
- **WebUI/API:** `http://localhost:7860` (o `http://192.168.12.7:7860` desde Fumilinux)
- **Outputs:** `/home/kelsielinux/apps/sdcpp/outputs/`
- **Modelos:** `/home/kelsielinux/apps/sdcpp/models/checkpoints/`

## Modelos instalados (stack reconstruido 2026-05-30)

| Archivo | Tamaño | Flag | Especialidad |
|---|---|---|---|
| `NoobAI-XL-v1.1.safetensors` | 6.7 GB | `noobai` | Likeness anime + NSFW Danbooru reforzado (fork Illustrious) |
| `animagine-xl-4.0-opt.safetensors` | 6.5 GB | `animagine` | Anime SFW puro refinado |
| `waiNSFWIllustrious_v14.safetensors` | 6.5 GB | `wai` | Mix NSFW masivo (Pony+Illustrious blend) |
| `ponyRealism_v22MainVAE.safetensors` | 6.7 GB | `ponyrealism` | Pony con anatomía mejorada |

**⚠️ Modelos REMOVIDOS — NO descargar de nuevo:**
- **2026-05-30**: `PonyDiffusionV6XL.safetensors` e `Illustrious-XL-v1.0.safetensors` — reemplazados por los 4 forks superiores arriba
- **2026-05-29**: `PonyDiffusionV6XL-Q5_0.gguf` y `MeinaHentai-baked-VAE-Q5_0.gguf` — GGUF Q5_0 en SDXL = ruido puro con sd.cpp
- **2026-05-29**: `MeinaHentai-baked-VAE.safetensors` — fallback SD 1.5 sin uso
- **2026-05-29**: `boring_sdxl_v1.safetensors` — producía PNG blanco determinista con Pony en prompts complejos

## LoRAs disponibles (en `~/apps/sdcpp/models/loras/`)

| LoRA | Tamaño | Propósito |
|---|---|---|
| `perfect_hands_v2.safetensors` | 436M | Manos/dedos |
| `sdxl_detail.safetensors` | 163M | Detalles finos |
| `smooth_anime_style_xl.safetensors` | 218M | Estilo anime suave |

Aplicar con sintaxis `<lora:nombre:peso>` en prompt (sd-server ya tiene `--lora-model-dir` configurado). Pesos sugeridos: 0.7 / 0.4 / 0.3.

## Métricas reales medidas (2026-05-27)

| Modelo | Resolución | Steps | Sampler | Tiempo |
|---|---|---|---|---|
| MeinaHentai Q5_0 + FA | 512×512 | 20 | dpm++2m | ~1m35s |
| MeinaHentai Q5_0 + FA | 512×768 | 28 | dpm++2m | ~3m45s |
| PonyDiffusion fp16 + FA + VAE-tiling | 768×768 | 20 | dpm++2m | ~7m21s |
| PonyDiffusion fp16 + FA + VAE-tiling | 768×768 | 20 | dpm++2m | ~9m14s (con prompt largo, 35+ tags) |
| PonyDiffusion fp16 + FA + VAE-tiling | 832×1216 | 30 | dpm++2mv2 | ~8m22s |

**Resolución máxima confirmada:** 832×1216 (vertical SDXL nativo) NO hace OOM con `--vae-tiling`. 1024×1024 sí falla en VAE decode incluso con tiling.

**Hallazgo:** `dpm++2mv2` converge más eficiente que `dpm++2m` — más steps en menos tiempo total. Usar por defecto.

## Settings por modelo

### PonyDiffusion V6 XL (SDXL) — configuración probada

```bash
# sd-cli
build/bin/sd-cli \
  -m models/checkpoints/PonyDiffusionV6XL.safetensors \
  --backend vulkan0 --fa --vae-tiling \
  -p "PROMPT" -n "NEGATIVE" \
  -W 768 -H 768 --steps 20 --cfg-scale 7.0 \
  --sampling-method dpm++2m --scheduler karras --clip-skip 2 \
  -o output.png

# sd-server (API en :7860)
./launch.sh  # usa Pony por defecto
```

**Prompt format Pony (OBLIGATORIO):**
- Positive: empezar con `score_9, score_8_up, score_7_up, source_anime, rating_explicit,` + prompt real
- Negative: empezar con `score_1, score_2, score_3, source_furry,` + negative real
- Sin estos score tags la calidad cae significativamente
- ⚠️ **NO usar `boring_sdxl_v1` en negative** — el 2026-05-29 se descubrió que produce PNG blanco determinista de 35507B con Pony en prompts complejos a 50+ steps. Archivo sigue en `/models/embeddings/` pero no referenciar.

### Illustrious XL v1.0 (SDXL) — configuración probada

```bash
# Cambiar modelo
~/projects/ia-gen/scripts/switch-model.sh illustrious

# Prompt format Danbooru puro (sin score tags)
# Positive: (masterpiece:1.2), (best quality:1.2), highly detailed, [personaje], [contenido], anime style, 2d
# Negative: (worst quality:1.4), (low quality:1.4), deformed, ..., western style

# Tags multipalabra con underscore: cum_in_mouth, sleep_sex, looking_at_viewer
```

**Nota Illustrious warmup:** primera generación tras switch tarda ~33 min. Las siguientes ~13 min warm.

## Trampas confirmadas

1. **🔴 Múltiples sd-server saturan VRAM.** Si hay 2 procesos (pasaba con `sdcpp.service Restart=on-failure` + `pkill -9`), ambos cargan Pony 6.8GB en VRAM 4GB → saturación y requests cuelgan silenciosos. **Service ahora `disabled` desde 2026-05-29.** Verificar `pgrep -c sd-server` = 1 antes de cada batch.
2. **🔴 PNG de ~34KB = imagen blanca / no guardada.** Era el síntoma con 8GB RAM pre-upgrade: VAE decode fallaba silenciosamente sin escribir error. `gen.sh` reescrito el 2026-05-29 ahora avisa con WARN si PNG < 50KB y aborta con exit≠0 si no se guardó.
3. **🔴 Error silencioso en gen.sh pre-fix:** `curl` sin `--max-time`, sin chequeo de HTTP status, sin chequeo de response JSON. Si sd-server colgaba o devolvía error, gen.sh seguía y reportaba "Sync OK" tomando una imagen vieja del chain anterior con `ls -t | head -1`. Fix aplicado 2026-05-29.
4. **ROCm HIP crash en kernel 6.17 — PERMANENTE.** Solución: Vulkan.
5. **Kernel 6.8 destruyó el equipo** → permanentemente descartado.
6. **SDXL Q5_0 GGUF genera ruido puro.** Solo safetensors fp16. GGUFs ya borrados.
7. **PonyDiffusion: prediction mode** se autodetecta eps. NO forzar `--prediction v`.
8. **VAE OOM a 1024×1024** incluso con `--vae-tiling`. Máximo: 832×1216 vertical o 768×768 cuadrado.
9. **`--schedule` (incorrecto) → usar `--scheduler`** en sd-cli.
10. **Tras reboot, sd-server queda en estado Vulkan inválido** — `ErrorDeviceLost`. Kill + relaunch.
11. **gen.sh original abría visor `eog` automático** — quitado 2026-05-27.
12. **(OBSOLETO post-upgrade) Llama3.2:3b no cargaba en Torre 1** con sd-server + Open WebUI + Firefox activos por RAM 8GB. Ahora con 15GB sí carga.

## Prompt NSFW probado — PonyDiffusion, composición sexual explícita

```
positive: score_9, score_8_up, score_7_up, source_anime, rating_explicit, (masterpiece), (best quality), 1girl, 1boy, mature female, long black hair, kneeling, looking up at viewer, from above, male pov, fellatio, oral, penis in mouth, penis from above, cum dripping, saliva, blush, detailed face, detailed eyes, indoors, soft lighting

negative: score_1, score_2, score_3, source_furry, (worst quality), (low quality), deformed, malformed, bad anatomy, extra limbs, extra hands, extra fingers, fused fingers, blurry, watermark, signature, text, censored, jpeg artifacts

settings: 768×768, 20 steps, CFG 7.0, DPM++ 2M Karras, clip_skip 2
```

## Modelos a considerar (no descargados)

| Modelo | URL HuggingFace | Tamaño | Nota |
|---|---|---|---|
| `AnimagineXL 3.1` | `cagliostrolab/animagine-xl-3.1` | ~6.4 GB | SDXL anime limpio (no requiere score tags) |
| `MeinaV10` | `Meina/MeinaMix` | 3.3 GB | SD 1.5 general, limpio |
| `CounterfeitV3` | `gsdf/Counterfeit-V3.0` | 4 GB | SD 1.5, estilo suave |
