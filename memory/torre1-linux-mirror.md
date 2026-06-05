# Torre 1 — Linux Setup completado 2026-05-26

## Hardware Torre 1
- GPU: AMD RX 570 8GB (Polaris, GFX8 / gfx803)
- OS: Ubuntu 24.04 LTS (instalado 2026-05-26)
- Uso: AI generation local + gaming con Proton

## Acceso remoto

### SSH
- IP LAN: 192.168.12.7, puerto 22
- IP Tailscale: 100.67.216.43
- Usuario principal: `kelsielinux` (NO es `kelsie` — trampa crítica)
- Usuario root: también tiene acceso SSH con la misma clave
- Clave: `~/.ssh/id_ed25519` de Fumilinux → autorizada en Torre 1 (verificado 2026-05-28 — ambas claves coinciden, SSH funcional)
- `kelsielinux` tiene sudo NOPASSWD

**Comando exacto (Tailscale — funciona desde cualquier red):**
```
ssh -o ConnectTimeout=10 kelsielinux@100.67.216.43
```
**Alternativa LAN (misma red):**
```
ssh kelsielinux@192.168.12.7
```
**Trampas frecuentes:**
- Usuario es `kelsielinux`, NO `kelsie` — da "Permission denied" sin error claro si usas el nombre incorrecto
- La clave `~/.ssh/id_ed25519` debe existir en Fumilinux
- Clave pública en Torre 1: `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHz5ABNXYXkMtKx6KvpNJhElQnDFHUdXnSudRlAykDL4 fumikatsu.koichi@gmail.com`
- Si falla con "Permission denied (publickey)": regenerar con `ssh-copy-id kelsielinux@100.67.216.43` (requiere acceso físico o contraseña temporal)

### Tailscale (VPN)
- Instalado en Torre 1 y Fumilinux (2026-05-26)
- Cuenta: fumikatsu.koichi@gmail.com
- Torre 1: 100.67.216.43 | Fumilinux: 100.116.50.47
- Funciona desde cualquier red

### Escritorio remoto (GNOME Remote Desktop + RDP)
- Puerto: **3389** (verificado 2026-06-04)
- Credenciales: kelsielinux / Fumi0926
- Certificado TLS en `~/.local/share/gnome-remote-desktop/`
- Comando: `xfreerdp /v:100.67.216.43:3389 /u:kelsielinux /p:Fumi0926 /cert:ignore /dynamic-resolution +clipboard`
- Lanzador en Fumilinux: `~/.local/share/applications/torre1-rdp.desktop` (actualizar puerto)
- **TRAMPA:** AnyDesk no funciona como host en Wayland
- Remmina: perfil "Torre Linux" en Fumilinux actualizado a `100.67.216.43:3389` (verificado 2026-06-04)
- Lanzador `.desktop` (`torre1-rdp.desktop`): usa xfreerdp, actualizado a puerto 3389 (2026-06-04). Si el ícono no abre nada tras editar el .desktop, reiniciar GNOME Shell: `Alt+F2 → r → Enter`

**TRAMPA:** El usuario en Torre 1 es `kelsielinux`, no `kelsie`. Todos los paths son `/home/kelsielinux/`. Los servicios systemd y lanzadores .desktop fueron corregidos con sed al migrar.

## Stack instalado (2026-05-26)

### Core
- Claude Code 2.1.150 (Node.js 24.15.0)
- Docker CE 29.1.3 + docker-compose-plugin + buildx
- n8n (docker, puerto 5678, volumen n8n_data)
- Ollama 0.24.0 + modelo llama3.2:3b
- MCP ollama-mcp configurado en ~/.claude.json

### AI / GPU
- **ROCm/HIP REMOVIDO 2026-05-27** — stack completo (17 GB, 60+ paquetes, /opt/rocm-6.4.0, /opt/amdgpu) purgado vía `amdgpu-install --uninstall`. Inservible con kernel 6.17 (segfault hipGetDeviceCount). Reemplazado por Vulkan/RADV (parte de Mesa estándar).
- Stack activo: Mesa 25.2.8 + RADV (Vulkan 1.4.318) — driver `radv`, device `AMD Radeon RX 570 Series (RADV POLARIS10)`
- `HSA_OVERRIDE_GFX_VERSION` removido de ~/.bashrc y ~/.profile (ya no aplica)
- `libgl1-amber-dri` (Mesa 21 legacy) también purgado
- kelsielinux en grupos render y video

### Apps apt
vlc, ffmpeg, mpv, rhythmbox, flameshot, gnome-screenshot, imagemagick,
remmina (+rdp/vnc/secret), thunderbird, yt-dlp, aria2,
7zip, p7zip-full, unrar, cabextract, copyq, transmission-gtk, deja-dup,
clamav (+daemon +freshclam), language-pack-es/ja, hunspell-es/en,
python3.12-venv, cmake, meson, ninja-build, autoconf, automake,
mesa-utils, xdotool, xclip, file-roller, simple-scan, gnome-calendar,
wine64 9.0, wine32:i386, winetricks, lutris 0.5.14,
gnome-shell-extension-manager, pavucontrol, vulkan-tools, clinfo,
zoom 7.x, anydesk 6.x, parsec, java 21 (openjdk-21-jre),
rclone v1.74.2, flatpak, snapd

### Apps Flatpak
- EasyEffects 8.2.4 + preset microfono-limpio copiado
- Sweet Home 3D 7.5

### Snaps
discord, steam, blender 5.1.2, code (VS Code), godot-4 4.6.1,
nextcloud, sublime-text, cups, firefox, thunderbird

### Servicios systemd (usuario)
- suwayomi-server.service — activo (Java 21 + JAR en ~/apps/suwayomi/)
- vector.service — habilitado (venv pendiente de instalar dependencias)
- voice-notifier.service — habilitado (idem)
- rclone-gdrive.service — habilitado (OAuth pendiente)

### Herramientas de seguridad
- vt-cli 1.3.1 + ~/.vt.toml (API key copiada desde Fumilinux)
- scan / scan-downloads / scan-home en ~/.local/bin/

## Archivos sincronizados desde Fumilinux
- ~/projects/ (4.3GB), ~/apps/, ~/.config/systemd/user/
- ~/.claude/ + ~/.claude.json, ~/.local/share/applications/ (corregidos), ~/.local/share/icons/
- ~/.local/bin/, ~/memoryClaude-main/, ~/.config/rclone/, ~/.var/app/easyeffects/

## Git configurado (2026-05-26)
- user.name / user.email configurados
- SSH key de GitHub activa (misma que Fumilinux, copiada a ~/.ssh/id_ed25519)
- `git pull` y `git push` funcionando desde `~/memoryClaude-main`

## Incidentes conocidos
- **GPU reset amdgpu (2026-05-26):** RX 570 hizo reset automático en primer arranque → pantalla se congeló → sistema se recuperó solo. Normal con GFX8 + ROCm. No es daño de hardware.
- **Nextcloud snap desactivado:** loop infinito de errores PHP saturando CPU. `sudo snap disable nextcloud`. Torre 1 no es servidor Nextcloud.
- **GRUB_DEFAULT trap:** SD.Next pidió cambiar a kernel 6.8 para ROCm HIP. Revertir siempre a `GRUB_DEFAULT=0` + `sudo update-grub`. Kernel estable: 6.17.0-29-generic.

## Pendiente manual
| Item | Acción |
|---|---|
| n8n workflows | Exportar desde Fumilinux localhost:5678 → importar en 192.168.12.7:5678 |
| Vector venv | `cd ~/voice-assistant && python3 -m venv .venv && .venv/bin/pip install -r requirements.txt` |
| **Upgrade hardware** | Ryzen 5 5500 + 16 GB DDR4 (canibalizar de otra torre con misma board). Resuelve cuello RAM/swap definitivamente. |

**OBSOLETO:** Kernel downgrade a 6.8 ya no aplica — ROCm/HIP descartado, Vulkan funciona con kernel 6.17 mainline.

**rclone Google Drive:** RESUELTO (2026-05-27) — OAuth activo, FUSE montado en ~/GoogleDrive, sincronizando.

## Optimizaciones de sistema (2026-05-27)
- `vm.swappiness=10` persistente en `/etc/sysctl.d/99-performance.conf`
- CPU governor `performance` persistente vía `/etc/systemd/system/cpu-performance.service` (habilitado)
- Servicios desactivados: `anydesk` (inútil en Wayland), `ModemManager` (sin modem), `avahi-daemon` (mDNS innecesario), `snap.cups.cups-browsed`
- **Nota:** Torre 1 corre GNOME en **Wayland** (no Xorg como Fumilinux). AnyDesk no funciona como host.
- Ryzen 5 2400G, 7.7GB DDR4 2400MHz. Con Steam activo: swap al 100% — cerrar Steam para liberar ~600MB.
- **RAM OOM (2026-05-27):** Con Firefox + 2×Claude Code + Ollama + Open WebUI + n8n + GNOME, la RAM se agota y el swap llega al 100% → OOM killer mata procesos. Solución: parar Ollama/Open WebUI/n8n cuando no se usen (`sudo systemctl stop ollama && sudo docker stop open-webui n8n`). `drop_caches` también libera ~1.2 GB de cache de disco sin riesgo. `swapoff -a` NO es viable con 3+ GB en swap y solo 2 GB libres — se mata solo.

## SD.Next — ELIMINADO 2026-05-27
- `/home/kelsielinux/apps/sdnext/` borrado. Reemplazado por `stable-diffusion.cpp` con backend Vulkan.
- Stack ROCm/HIP que SD.Next requería también purgado (ver sección AI/GPU arriba).
- Pony Diffusion V6 XL corre en sd-server vía Vulkan a 768×768 / 20 steps en ~7min.
- Ver detalles en `memory/ai-image-local.md`.
