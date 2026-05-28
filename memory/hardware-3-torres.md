# Hardware: 3 torres físicas de Koichi

Verificado 2026-05-28 vía dmidecode + lspci + vulkaninfo.

## Torre Oficina (primary, kelsielinux user)

- **CPU:** Ryzen 5 2400G (Zen, 4c/8t, Vega 11 iGPU)
- **Placa:** Gigabyte B450M DS3H V2, BIOS F63 (2022-08-03) — soporta Zen 3
- **RAM slots:** 4 totales, **2 ocupados** (DIMM 1 Channel A+B). DIMM 0 vacíos.
- **RAM:** 8GB DDR4-2400 (2×4GB mixto: ADATA + Unknown)
- **GPU:** ASUS RX 570 4GB (Polaris/gfx803, RADV/Vulkan only)
- **PSU:** Thermaltake Smart 500W 80+ White
- **Storage:** SanDisk SN570 2TB NVMe
- **OS:** Ubuntu 24.04, kernel 6.17

**Estado limitación crítica:** ROCm/HIP roto en kernel 6.17 (descartado permanentemente). Solo Vulkan vía sd.cpp. VRAM 4GB obliga a `--fa --vae-tiling` para SDXL, techo de generación 832×1216, ~8min/imagen, swap heavy a RAM/disco.

## Torre Casa (gaming, no usa para trabajo)

- **CPU:** Ryzen 5 5500 (Zen 3, 6c/12t, **sin iGPU**)
- **RAM:** 24GB (config DIMMs pendiente verificar)
- **GPU:** RX 6500 XT 4GB (RDNA 2/gfx1034, bus 64-bit, PCIe 4.0 ×4)
- **PSU:** 850W

## Torre AZC (empresa AZC Legal, NO TOCAR)

- **CPU:** Ryzen 5 3400G (Zen+, 4c/8t, Vega 11 iGPU)
- **RAM:** 16GB
- **GPU:** RX 6500 XT 4GB
- **PSU:** 750W

## Plan acordado: swap CPU+RAM Casa ↔ Oficina

Decisión del 2026-05-28: intercambiar CPU+RAM entre Casa y Oficina para upgradear rig primario de AI sin gasto. Casa pierde poder pero solo se usa para juegos.

**Post-swap:**
- Oficina: 5500 + 24GB + RX 570 + 500W
- Casa: 2400G + 8GB + RX 6500 XT + 850W

**Ganancias en Oficina:** generación 832×1216 baja de ~11min a ~7-8min (sin swap), Ollama + sd-server + Firefox coexisten, LoRA SD 1.5 local posible.

**Sigue siendo cuello:** GPU 4GB. SDXL LoRA local imposible hasta upgrade GPU futuro.

## Próximo upgrade GPU (futuro, presupuesto $200-280)

Cuando sea, prioridad alta: GPU 12GB+ para Torre Oficina (post-swap, con 850W disponible si después se swap también la PSU). PSU 500W aguanta hasta RX 6700 XT 230W.

**Opciones:**
- RX 6700 XT 12GB usada (~$220) — sweet spot AMD, ROCm oficial
- RTX 3060 12GB usada (~$220) — sweet spot NVIDIA, CUDA más fácil
- RX 7600 XT 16GB nueva (~$330) — RDNA 3, más VRAM

Koichi prefiere mantenerse AMD si viable.

## NO aplicables (descartados)

- **Quadro K4000** (en algún cajón): chatarra para AI. CC 3.0 obsoleto, 3GB, Kepler 2012.
- **RX 6500 XT como upgrade Oficina**: downgrade efectivo. Bus 64-bit + PCIe 4.0 ×4 la hace más lenta que RX 570 para SD por swap heavy.
