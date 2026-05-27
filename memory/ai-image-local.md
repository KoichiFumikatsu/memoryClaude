# AI image generation — Estado y aprendizajes

**Workspace de proyecto:** `~/projects/ia-gen/` — README + symlinks a `outputs/` y `models/` + `scripts/` con wrappers + `prompts/` + `video/` (placeholder futuro). Ver `~/projects/ia-gen/README.md` y `~/projects/ia-gen/CLAUDE.md` (este último se autocarga al abrir Claude Code en el workspace, contiene config funcional default).

**Launcher Desktop:** `~/Desktop/ia-gen-claude.desktop` — abre terminal en `~/projects/ia-gen/` y arranca `claude`. Junto a `sdcpp-webui.desktop` (port 7860) y `open-webui.desktop` (port 3000).

**Estado actual Torre 1 (2026-05-27):** `stable-diffusion.cpp` compilado con backend Vulkan (RADV). ROCm/HIP descartado — crash irrecuperable en kernel 6.17. Stack activo: Vulkan + PonyDiffusion V6 XL safetensors fp16.

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

## Modelos instalados

| Archivo | Tamaño | Estado | Uso |
|---|---|---|---|
| `PonyDiffusionV6XL.safetensors` | 6.5 GB | **ACTIVO — modelo principal** | SDXL, alta calidad, NSFW explícito |
| `PonyDiffusionV6XL-Q5_0.gguf` | 2.9 GB | **ROTO — genera ruido** | Descartado |
| `MeinaHentai-baked-VAE-Q5_0.gguf` | 1.6 GB | Funcional, calidad menor | SD 1.5 fallback rápido |
| `MeinaHentai-baked-VAE.safetensors` | 2.0 GB | Original SD 1.5 | Puede borrarse |

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

### MeinaHentai (SD 1.5) — fallback rápido

```bash
./launch.sh models/checkpoints/MeinaHentai-baked-VAE-Q5_0.gguf
# settings: 512×512, 20 steps, CFG 6, DPM++ 2M Karras, clip_skip 2
```

## Trampas confirmadas

1. **ROCm HIP crash en kernel 6.17 — PERMANENTE.** HSA funciona, HIP segfault. No hay workaround. Solución: Vulkan.
2. **Kernel 6.8 destruyó el equipo** → permanentemente descartado.
3. **SDXL Q5_0 GGUF genera ruido puro.** La conversión `sd-cli -M convert --type q5_0` funciona para SD 1.5 (MeinaHentai) pero produce GGUF inservible para modelos SDXL. Usar safetensors fp16 directo.
4. **PonyDiffusion: prediction mode.** Se autodetecta como eps-prediction desde el safetensors — correcto, NO forzar `--prediction v` (produce noise). El `--prediction v` solo genera ruido con este modelo.
5. **VAE OOM a 1024×1024** incluso con `--vae-tiling`. Máximo confirmado: 832×1216 vertical o 768×768 cuadrado.
6. **`--schedule` (incorrecto) → usar `--scheduler`** en sd-cli.
7. **Anatomía POV "disembodied penis"** — SD 1.5 no entiende la física espacial. Genera el eje en dirección incorrecta siempre. Pony (SDXL) maneja mejor composiciones complejas.
8. **Tras reboot del equipo, sd-server queda en estado Vulkan inválido** — proceso sigue vivo pero al llamar `/sdapi/v1/txt2img` devuelve `vk::Queue::submit: ErrorDeviceLost`. Solución: `kill <pid>` + `nohup ./launch.sh > /tmp/sdserver.log 2>&1 &`. Esperar ~8s a que cargue el modelo (6.5GB).
9. **gen.sh original abría visor `eog` automático** — quitado 2026-05-27 a pedido del usuario. Output queda solo en `outputs/`.
10. **Llama3.2:3b no genera prompts en Torre 1 cuando sd-server + Open WebUI + Firefox están activos** — RAM insuficiente (8GB total). Fallback: armar prompts manualmente o usar OpenAI/Groq/Gemini via Open WebUI.

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
