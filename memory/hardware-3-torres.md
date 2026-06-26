# Hardware de Koichi (actualizado 2026-06-26)

**Realidad actual: 2 equipos.** El esquema viejo de "3 torres" (Oficina/Casa/AZC) ya NO aplica
— ver histórico al final.

## Torre 1 — único desktop (verificado por SSH 2026-06-26)
Es lo que antes se llamaba "Torre Casa" (gaming). El swap CPU+RAM Casa↔Oficina **sí se hizo**:
la 2400G+8GB salió, entró la 5500. Quedó como el único desktop de Koichi.

- **CPU:** Ryzen 5 5500 (Zen 3, 6c/12t, **sin iGPU**)
- **RAM:** 16 GB (15 GiB usable)
- **GPU:** ASUS RX 570 **4 GB** (Polaris/Ellesmere, gfx803, `[1002:67df]`) — **RADV/Vulkan only**
- **OS:** Ubuntu 24.04.2, kernel 6.17 · usuario `kelsielinux` · Tailscale `100.67.216.43` / LAN `192.168.12.7`
- **Rol:** caja de generación de imágenes (sd.cpp Vulkan + dashboard la orquesta por SSH) Y, a futuro,
  daily driver + gaming en Windows.

**Limitación crítica:** ROCm/HIP descartado permanentemente (crash en kernel 6.17). Solo Vulkan vía
sd.cpp. **VRAM 4 GB** = el cuello real: obliga `--fa --vae-tiling`, hires es el pico de presión, y
no se puede generar + jugar a la vez (pelean por la misma VRAM → crash). Ver [[ia-gen-dashboard]].

**Plan OS (decidido 2026-06-26):** dual-boot Windows (daily + gaming) + Linux (sesiones de generación,
típicamente chains overnight). Koichi se cansó de pelear con Linux para uso diario. NO WSL2 (passthrough
Vulkan-compute a WSL2 en Polaris es inviable). El stack de ia-gen sigue intacto en la partición Linux;
el dashboard del portátil solo llega cuando Torre 1 está booteada en Linux. Alternativa rechazada por
costo/rework: generación en nube (RunPod).

## Fumilinux — portátil (equipo de trabajo)
Donde Koichi trabaja. Corre el dashboard `iagen-dashboard` (:8769) y orquesta Torre 1 por SSH.
Tailscale `100.116.50.47`. Sin GPU capaz de SDXL → no es alternativa de generación.

## Próximo upgrade GPU (futuro, presupuesto ~$200-280)
Prioridad alta: GPU 12GB+ para Torre 1 (PSU aguanta hasta ~230W). Koichi prefiere AMD si viable.
- RX 6700 XT 12GB usada (~$220) — sweet spot AMD, ROCm oficial
- RTX 3060 12GB usada (~$220) — sweet spot NVIDIA, CUDA más fácil
- RX 7600 XT 16GB nueva (~$330) — RDNA 3, más VRAM
Quadro K4000 = chatarra para AI (descartada). RX 6500 XT = downgrade vs RX 570 para SD (bus 64-bit).

## Histórico (ya NO aplica)
Hasta ~2026-05-28 había 3 torres: **Oficina** (2400G+8GB+RX 570, primary AI), **Casa**
(5500+24GB+RX 6500 XT, gaming) y **AZC** (empresa, no tocar). Se acordó swap CPU+RAM Casa↔Oficina
para upgradear el rig de AI sin gasto. Resultado consolidado a 2026-06: una sola Torre 1
(5500+16GB+RX 570). Las otras dos ya no están en posesión de Koichi para uso personal.
