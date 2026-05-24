# AI image generation local — SD.Next + OpenVINO en Fumilinux

**Operativo desde:** 2026-05-23
**Path:** `/home/kelsie/projects/ai-image/sdnext/`
**Backend:** OpenVINO FX sobre Intel Iris Xe iGPU (`Device: active="GPU"`)
**UI:** http://127.0.0.1:7860/ (solo localhost; `--listen` solo si se necesita LAN)

## Comando de launch

```bash
cd /home/kelsie/projects/ai-image/sdnext && \
OPENVINO_DEVICE=GPU OPENVINO_TORCH_BACKEND_DEVICE=GPU \
./venv/bin/python launch.py --use-openvino --debug
```

**No usar `--lowvram` ni `--medvram` con OpenVINO.** Ambos rompen el backend FX.

## Configuración crítica (debe mantenerse)

1. **Branch `dev` de SD.Next** (`master` falla con torch 2.11.0+cpu / OpenVINO 2026.1.0).
2. **Patch local en `modules/sd_models.py:848`** — no upstream:
   ```python
   "low_cpu_mem_usage": not shared.cmd_opts.use_openvino,
   ```
   Tras cualquier `git pull` verificar que siga aplicado.
3. **`config.json`** raíz del repo:
   ```json
   {
     "gradio_theme": "Default",
     "openvino_devices": ["GPU"],
     "openvino_disable_model_caching": false,
     "openvino_disable_memory_cleanup": false
   }
   ```

## Modelos descargados

| Archivo | Tipo | Cabe en Fumilinux | Notas |
|---|---|---|---|
| `CounterfeitV3.safetensors` | SD 1.5 | sí | probado OK |
| `AOM3A1B.safetensors` | SD 1.5 | sí | sin probar |
| `AnimagineXL3.safetensors` | SDXL | **no** | causó OOM kill |
| `PonyDiffusionV6XL.safetensors` | SDXL | **no** | no probado por riesgo OOM |
| `models/VAE/kl-f8-anime2.ckpt` | VAE anime | — | disponible |
| `models/embeddings/EasyNegative.safetensors` | embedding | — | usar en negative |

## Performance medida (CounterfeitV3, 512x768, 20 steps, Euler a)

- Primera generación (compila modelo a IR cache): **~110 s**
- Subsecuentes con cache: ~30-50 s estimado
- Velocidad: ~0.19 it/s

## API: forzar modelo SD 1.5 en cada request

SD.Next persiste el último modelo seleccionado de la UI entre reinicios. Si la UI quedó apuntando a SDXL (Animagine/Pony), siguiente arranque carga ese → cualquier txt2img API genera contra SDXL → OOM kill. **Incluir siempre en el body de la API**:

```json
"override_settings": {
  "sd_model_checkpoint": "CounterfeitV3",
  "sd_vae": "kl-f8-anime2.ckpt",
  "CLIP_stop_at_last_layers": 2
},
"override_settings_restore_afterwards": false
```

`override_settings_restore_afterwards: false` deja CounterfeitV3 cargado tras la generación.

## Trampas (no repetir)

1. **`--lowvram` rompe OpenVINO**: `accelerate.modeling._load_state_dict_into_meta_model:373` llama `param_cls(new_value, requires_grad=...)` pero el `param_cls` parcheado por OpenVINO no acepta ese kwarg → `TypeError`.
2. **`--medvram` rompe OpenVINO**: `model_cpu_offload` deja `FakeTensor` en el grafo, OpenVINO no los puede convertir a IR (`AssertionError: FakeTensor detected` en `openvino/frontend/pytorch/utils.py:68`).
3. **`low_cpu_mem_usage=True`** (default de diffusers) provoca el mismo FakeTensor → por eso el patch lo desactiva. Sin patch, master y dev fallan idéntico.
4. **SDXL no cabe en Fumilinux**: 6.4 GB modelo + activaciones + python + OS + navegador requiere ~12-14 GB sin offload. iGPU comparte RAM con el sistema. Primer intento murió por OOM kill del kernel (visible en `journalctl --user --since "..." | grep -i oom`).
5. **`--listen` expone también la IP pública del router** (no solo LAN). Solo añadir si necesario.
6. **El cómputo siempre corre en el host de SD.Next**, no en el cliente del navegador. Abrir UI desde Torre no usa la RX 570 si SD.Next corre en Fumilinux.

## Salidas y cache

- Imágenes: `outputs/text/NNNNN-YYYY-MM-DD-<modelo>.jpg`
- Cache de modelos compilados (IR): `cache/blob/` + `cache/model/` (puede crecer 10-15 GB)

## Decisiones pendientes

- **SDXL solo en hardware con GPU dedicada**. Esperar migración Torre 1 (RX 570) a Linux con ROCm + `HSA_OVERRIDE_GFX_VERSION=9.0.0`; alternativa intermedia: SD.Next en Torre 1 Windows con `--use-directml` o `--use-zluda` para AMD.
- No actualizar SD.Next dev sin verificar antes que el patch siga aplicado.
