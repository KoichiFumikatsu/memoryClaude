# AI image generation — Estado y aprendizajes

**Estado actual:** REMOVIDO de Fumilinux (`/home/kelsie/projects/ai-image/` borrado 2026-05-24, liberó 36 GB).
**Razón:** Intel Iris Xe iGPU + OpenVINO 2026.1 + torch 2.11 + diffusers 0.39-dev solo soporta txt2img básico. Hires fix, detailer, img2img e inpaint disparan bug `FakeTensor` en OpenVINO FX backend. Sin posibilidad de iterar/refinar imágenes.
**Plan:** Migrar a Torre 1 (AMD RX 570 8GB) — Linux con ROCm `HSA_OVERRIDE_GFX_VERSION=9.0.0`, o Windows con `--use-directml`/`--use-zluda`.

---

## Para Torre 1: aprendizajes técnicos (no repetir errores)

### SD.Next branch

En Torre 1 con DirectML/ROCm/ZLUDA, usar **branch `master`**. El bug FakeTensor que obligó a usar `dev` en Fumilinux es específico de `openvino_fx`, no aparece en otros backends.

### Modos `--lowvram`/`--medvram`

- `--lowvram`: `accelerate.modeling._load_state_dict_into_meta_model:373` llama `param_cls(new_value, requires_grad=...)`. Falla solo en backends que parchean `Parameter` (OpenVINO). OK en CUDA/ROCm/DirectML.
- `--medvram`: `model_cpu_offload` deja FakeTensors. OK fuera de OpenVINO FX.
- Torre 1 con 8 GB VRAM dedicada probablemente no necesita offload para SD 1.5 ni SDXL. Si requiere: `--medvram` antes que `--lowvram`.

### Modelos para re-descargar en Torre 1

| Archivo | URL HuggingFace | Tamaño | Uso |
|---|---|---|---|
| `MeinaHentai - baked VAE.safetensors` | `Meina/MeinaMix` | 2 GB | NSFW + anatomía limpia (probado y mejor que CounterfeitV3) |
| `Meina V10 - baked VAE.safetensors` | `Meina/MeinaMix` | 3.3 GB | General anime/manga limpio |
| `AnimagineXL3` | `cagliostrolab/animagine-xl-3.0` | 6.4 GB | SDXL anime (ahora viable con GPU dedicada) |
| `PonyDiffusionV6XL` | `AstraliteHeart/pony-diffusion-v6` | 6.5 GB | SDXL alternativo |
| `CounterfeitV3` | `gsdf/Counterfeit-V3.0` | 4 GB | SD 1.5 base, opcional |
| `AOM3A1B` | `WarriorMama777/OrangeMixs` | 2 GB | SD 1.5 alternativo |

### Embeddings (descargar siempre, ~300 KB total)

- `bad-hands-5.pt` (`yesyeahvh/bad-hands-5`)
- `bad_prompt_version2.pt` (`datasets/Nerfgun3/bad_prompt`)
- `ng_deepnegative_v1_75t.pt` (`lenML/DeepNegative`)
- `EasyNegative.safetensors` (estándar comunidad)

VAE para SD 1.5 sin baked-VAE: `kl-f8-anime2.ckpt` (`hakurei/waifu-diffusion-v1-4/vae`)

### Prompt template NSFW probado (MeinaHentai, 2026-05-24)

Composición POV: chica arrodillada mirando arriba, solo se ve el miembro masculino disembodied + cum en cara/boca.

```
prompt: (masterpiece:1.2), (best quality:1.2), (ultra-detailed:1.1), manga illustration, clean lineart, vibrant colors, very aesthetic, 1girl, solo, mature female, [traits...], kneeling on floor, looking up at viewer, head tilted up, mouth open, tongue out, (large penis:1.2), penis on face, disembodied penis, cum, (cum on face:1.3), (cum in mouth:1.3), facial, cum on tongue, cum string, blush, saliva, after fellatio, indoors, soft lighting, depth of field, detailed face, detailed eyes, pov perspective

negative: EasyNegative, ng_deepnegative_v1_75t, bad_prompt_version2, bad-hands-5, (worst quality:1.4), (low quality:1.4), (extra hands:1.4), (extra arms:1.4), (extra fingers:1.4), (extra limbs:1.4), (floating limbs:1.4), fused fingers, malformed limbs, deformed, blurry, watermark, signature, text, censored, mosaic censoring, bar censor, jpeg artifacts, lowres, 1boy, multiple boys, male focus, male body, male torso, male legs

settings: 512x768, 28 steps, CFG 6, DPM++ 2M + Karras, sd_vae None (baked-in MeinaHentai), CLIP skip 2
```

### API settings críticos

```json
"override_settings": {"sd_model_checkpoint": "<MODELO>", "sd_vae": "None", "CLIP_stop_at_last_layers": 2}
"override_settings_restore_afterwards": false
```

`override_settings_restore_afterwards: false` evita que SD.Next vuelva al último modelo seleccionado en UI (que puede ser SDXL pesado y reventar OOM en arranque siguiente).

### Tags para estilo

- B/N manga: `monochrome, manga style, screentone, dot pattern shading, ink lineart, manga panel, halftone`
- Color manga: `manga illustration, clean lineart, vibrant flat colors, cel shading, soft pastel manga style`
- Anatomía limpia: `realistic anime eyes, detailed eyes, defined pupils, natural hair flow, natural tongue, anatomically correct`
- Artistas referencia Koichi: Wagashi, Kamuo (NSFW manga, soft pastel + clean lineart). Tags `by wagashi, by kamuo`. Efectividad depende del modelo. Si no funciona: buscar LoRAs específicos en CivitAI/HuggingFace.

### Trampas Torre 1

1. **RX 570 (Polaris GFX8) no soporta ROCm reciente directamente**. Workaround: `HSA_OVERRIDE_GFX_VERSION=9.0.0` en env. Sin ese flag → SD detecta solo CPU.
2. **DirectML en Windows** funciona out-of-the-box con `--use-directml`. Más lento que ROCm pero estable.
3. **ZLUDA en Windows** (traduce CUDA→AMD) lo más rápido para AMD Windows si compila — requiere HIP SDK 5.7+.
4. Torre 1 actualmente con gachas/gaming Windows. Antes de migrar a Linux, evaluar dual-boot o instalar SD.Next en Windows (DirectML) primero para validar workflow sin commitearse al OS.

### Métricas de referencia (Fumilinux Iris Xe, para comparar después)

- SD 1.5 (CounterfeitV3 / MeinaHentai) 512x768 28 steps: **110-135 s por imagen** (OpenVINO compile cached)
- Velocidad: ~0.19-0.28 it/s
- SDXL: OOM kill consistente (no cabe en 24 GB RAM compartida)

En Torre 1 con RX 570 8GB VRAM dedicada, esperar:
- SD 1.5 512x768: ~15-30 s
- SDXL 1024x1024: ~60-90 s
- Hires fix + detailer accesibles
