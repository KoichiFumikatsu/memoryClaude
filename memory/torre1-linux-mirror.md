# Torre 1 — Linux Setup + Mirror desde Fumilinux

## Hardware Torre 1
- GPU: AMD RX 570 8GB (Polaris, GFX8)
- OS actual: Windows — candidata a Linux
- Uso objetivo: AI generation local + gaming con Proton

## OS elegido
Ubuntu 24.04 LTS — misma que Fumilinux para consistencia total de stack.

**Razón:** Consistencia con Fumilinux (mismos comandos, rutas, servicios systemd, Docker, scripts). El workaround ROCm para RX 570 (`HSA_OVERRIDE_GFX_VERSION=9.0.0`) está documentado principalmente en Ubuntu. Nobara descartada por sacar del ecosistema apt.

## Plan de mirror — 7 pasos

### Paso 1 — Instalar Ubuntu 24.04 + habilitar SSH en Torre 1
```bash
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
```

### Paso 2 — Exportar lista de paquetes desde Fumilinux
```bash
dpkg --get-selections | grep -v deinstall > ~/paquetes-fumilinux.txt
flatpak list --app --columns=application > ~/flatpaks-fumilinux.txt
```

### Paso 3 — Sincronizar por SSH (LAN) desde Fumilinux a Torre 1
```bash
# Reemplazar TORRE_IP con la IP local de Torre 1
rsync -avz --progress ~/projects/ kelsie@TORRE_IP:~/projects/
rsync -avz --progress ~/.config/ kelsie@TORRE_IP:~/.config/
rsync -avz --progress ~/.local/share/applications/ kelsie@TORRE_IP:~/.local/share/applications/
rsync -avz ~/.claude.json kelsie@TORRE_IP:~/
```

### Paso 4 — Restaurar paquetes en Torre 1
```bash
sudo dpkg --set-selections < ~/paquetes-fumilinux.txt
sudo apt-get dselect-upgrade -y
while read app; do flatpak install flathub "$app" -y; done < ~/flatpaks-fumilinux.txt
```

### Paso 5 — Docker + n8n + Ollama (reinstalar, no copiar)
```bash
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker kelsie

curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2:3b

sudo docker run -d --name n8n -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n n8nio/n8n
```

### Paso 6 — Servicios systemd
Los archivos `.service` ya se copian con el rsync de ~/.config/systemd/user/ en Paso 3.
```bash
systemctl --user daemon-reload
systemctl --user enable --now tlgames-pipeline tlgames-qa tlgames-versions
systemctl --user enable --now vector suwayomi
```

### Paso 7 — Claude Code
```bash
npm install -g @anthropic-ai/claude-code
# ~/.claude.json ya copiado en Paso 3
```

## Items que requieren configuración manual

| Item | Por qué | Acción |
|---|---|---|
| rclone Google Drive | Token OAuth es por máquina | `rclone config` de nuevo, autenticar con misma cuenta Google |
| EasyEffects preset | Ruta Flatpak distinta hasta instalar | Instalar Flatpak primero, luego copiar preset desde ~/.var/app/... |
| ROCm / HSA para RX 570 | Específico de hardware AMD GFX8 | Instalar ROCm + setear `HSA_OVERRIDE_GFX_VERSION=9.0.0` |
| GNOME keybindings/temas | Dependen del monitor/hardware | Se copian pero pueden necesitar ajuste visual |

## Estado
- Pendiente: Torre 1 sigue en Windows (2026-05-26)
- No ejecutado aún — documentado para cuando Koichi arranque la instalación
