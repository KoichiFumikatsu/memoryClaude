# AI image generation — Estado y aprendizajes

**Estado actual Torre 1 (2026-05-26):** SD.Next clonado, PyTorch instalado, **bloqueado por kernel 6.17 incompatible con ROCm 6.4**. Pendiente reboot con kernel 6.8.0-31 para continuar.

**Historia:** REMOVIDO de Fumilinux (`/home/kelsie/projects/ai-image/` borrado 2026-05-24, liberó 36 GB).
**Razón original:** Intel Iris Xe iGPU + OpenVINO 2026.1 + torch 2.11 + diffusers 0.39-dev solo soporta txt2img básico. Hires fix, detailer, img2img e inpaint disparan bug `FakeTensor` en OpenVINO FX backend. Sin posibilidad de iterar/refinar imágenes.

## Estado instalación Torre 1 (2026-05-26) — PARCIAL, pendiente reboot

- SD.Next clonado: `/home/kelsielinux/apps/sdnext/` (branch `master`)
- venv: `/home/kelsielinux/apps/sdnext/venv/`
- PyTorch instalado: `2.9.1+rocm6.3`
- Fix aplicado: `torch/lib/libamdhip64.so` → symlink a `/opt/rocm/lib/libamdhip64.so` (reemplaza bundled ROCm 6.3 que tenía `pthread_setaffinity_np` undefined)
- **BLOQUEADO:** `hipGetDeviceCount` segfault — ver sección "Trampas Torre 1 Kernel"

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

1. **RX 570 (Polaris GFX8) no soporta ROCm reciente directamente**. Workaround: `HSA_OVERRIDE_GFX_VERSION=9.0.0` en env. Sin ese flag → SD detecta solo CPU. Ya configurado en `~/.bashrc` y `~/.profile`.

2. **PyTorch wheel bundlea libamdhip64 incompatible con sistema.** El wheel ROCm 6.3 de pytorch.org incluye `torch/lib/libamdhip64.so` que tiene `pthread_setaffinity_np` undefined en Ubuntu 24.04. Fix: reemplazar con symlink al sistema: `ln -s /opt/rocm/lib/libamdhip64.so venv/lib/python3.12/site-packages/torch/lib/libamdhip64.so`. Ya aplicado.

3. **Kernel 6.17 incompatible con ROCm 6.4 en el path HIP init.** `rocminfo` y `rocm-smi` funcionan (HSA level OK). `hipGetDeviceCount` segfault con GDB stack: `libhsa-runtime64.so.1 → pthread_once → SIGSEGV` (después de cargar COMGR). El crash ocurre en el path de init que HIP usa pero `rocminfo` no. **Fix: boot con kernel 6.8.0-31-generic** (disponible en apt, es el target certificado de ROCm 6.4 para Ubuntu 24.04).
   ```bash
   sudo apt install linux-image-6.8.0-31-generic linux-headers-6.8.0-31-generic
   # Seleccionar en GRUB: Advanced options → 6.8.0-31-generic
   sudo reboot
   ```

4. **DirectML en Windows** funciona out-of-the-box con `--use-directml`. Más lento que ROCm pero estable.

5. **ZLUDA en Windows** (traduce CUDA→AMD) lo más rápido para AMD Windows si compila — requiere HIP SDK 5.7+.

### Pasos post-reboot para completar SD.Next

Después de confirmar que `hipGetDeviceCount > 0` con kernel 6.8:
```bash
cd /home/kelsielinux/apps/sdnext
source venv/bin/activate
# Verificar GPU primero:
python3 -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
# Lanzar SD.Next:
HSA_OVERRIDE_GFX_VERSION=9.0.0 python launch.py --backend rocm --listen --port 7860
```
Primer lanzamiento instala dependencias adicionales (~5-10 min). Descargar MeinaHentai primero para prueba rápida.

### Métricas de referencia (Fumilinux Iris Xe, para comparar después)

- SD 1.5 (CounterfeitV3 / MeinaHentai) 512x768 28 steps: **110-135 s por imagen** (OpenVINO compile cached)
- Velocidad: ~0.19-0.28 it/s
- SDXL: OOM kill consistente (no cabe en 24 GB RAM compartida)

En Torre 1 con RX 570 8GB VRAM dedicada, esperar:
- SD 1.5 512x768: ~15-30 s
- SDXL 1024x1024: ~60-90 s
- Hires fix + detailer accesibles
