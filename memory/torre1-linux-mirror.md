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
- Clave: `~/.ssh/id_ed25519` de Fumilinux → autorizada en Torre 1
- `kelsielinux` tiene sudo NOPASSWD

### Tailscale (VPN)
- Instalado en Torre 1 y Fumilinux (2026-05-26)
- Cuenta: fumikatsu.koichi@gmail.com
- Torre 1: 100.67.216.43 | Fumilinux: 100.116.50.47
- Funciona desde cualquier red

### Escritorio remoto (GNOME Remote Desktop + RDP)
- Puerto: 3389, credenciales: kelsielinux / Fumi0926
- Certificado TLS en `~/.local/share/gnome-remote-desktop/`
- Comando: `xfreerdp /v:100.67.216.43:3389 /u:kelsielinux /p:Fumi0926 /cert:ignore /dynamic-resolution +clipboard`
- Lanzador en Fumilinux: `~/.local/share/applications/torre1-rdp.desktop`
- **TRAMPA:** AnyDesk no funciona como host en Wayland
- **TRAMPA:** Remmina no conecta — usar xfreerdp directo

**TRAMPA:** El usuario en Torre 1 es `kelsielinux`, no `kelsie`. Todos los paths son `/home/kelsielinux/`. Los servicios systemd y lanzadores .desktop fueron corregidos con sed al migrar.

## Stack instalado (2026-05-26)

### Core
- Claude Code 2.1.150 (Node.js 24.15.0)
- Docker CE 29.1.3 + docker-compose-plugin + buildx
- n8n (docker, puerto 5678, volumen n8n_data)
- Ollama 0.24.0 + modelo llama3.2:3b
- MCP ollama-mcp configurado en ~/.claude.json

### AI / GPU
- ROCm 6.4 vía amdgpu-install — RX 570 detectada como gfx803
- `HSA_OVERRIDE_GFX_VERSION=9.0.0` en ~/.bashrc y ~/.profile
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

## Pendiente manual (4 items)
| Item | Acción |
|---|---|
| **Kernel downgrade** | `sudo apt install linux-image-6.8.0-31-generic` → reboot → seleccionar en GRUB → requerido para ROCm HIP |
| rclone Google Drive | `rclone config reconnect gdrive:` en Torre 1 |
| n8n workflows | Exportar desde Fumilinux localhost:5678 → importar en 192.168.12.7:5678 |
| Vector venv | `cd ~/voice-assistant && python3 -m venv .venv && .venv/bin/pip install -r requirements.txt` |

## SD.Next — Estado 2026-05-26 (parcial, bloqueado por kernel)

- Clonado en `/home/kelsielinux/apps/sdnext/` (master)
- venv en `apps/sdnext/venv/`, PyTorch 2.9.1+rocm6.3 instalado
- `torch/lib/libamdhip64.so` → symlink a `/opt/rocm/lib/libamdhip64.so` (fix pthread_setaffinity_np)
- Bloqueado: kernel 6.17 causa segfault en `hipGetDeviceCount` (HIP init path incompatible con ROCm 6.4)
- Ver detalles completos y pasos post-reboot en `memory/ai-image-local.md`
