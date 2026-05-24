# Google Drive vía rclone en Fumilinux

Activo desde 2026-05-24. No existe cliente oficial de Google Drive para Linux — esta es la alternativa CLI con FUSE.

## Configuración actual

- **rclone**: v1.74.2 (instalado vía script oficial `curl https://rclone.org/install.sh | sudo bash`)
- **Remote**: `gdrive` → fumikatsu.koichi@gmail.com (plan 100 GB Google One)
- **Mount point**: `~/GoogleDrive` (modo FUSE on-demand con vfs-cache)
- **Cache**: `~/.cache/rclone`, 10 GiB máximo, TTL 7 días, `--vfs-cache-mode full`
- **Logs**: `~/.cache/rclone/mount.log`
- **Service**: `~/.config/systemd/user/rclone-gdrive.service` (enabled + active, arranca al login)

## Comandos clave

```bash
systemctl --user status rclone-gdrive.service
systemctl --user restart rclone-gdrive.service
tail -f ~/.cache/rclone/mount.log
rclone about gdrive:
rclone listremotes
rclone config reconnect gdrive:
```

## Pendientes

- **Bisync**: configurar sync local bidireccional de carpetas específicas (frecuencia decidida: cada hora vía timer systemd). Carpetas a definir.
- **Client ID OAuth propio**: actualmente usa el client_id compartido de rclone (rate-limited por Google). Para uso intenso crear uno propio en Google Cloud Console. Sin urgencia mientras no aparezcan errores 403.

## Trampas conocidas

- `fusermount3` (no `fusermount`) en Ubuntu 24.04. El service usa `/bin/fusermount3 -u` en ExecStop.
- `Type=notify` requiere rclone reciente (≥1.59). v1.74.2 OK.
- El service es `--user`, no system. No requiere lingering (arranca al login interactivo).
- Si se pierde el OAuth: `rclone config reconnect gdrive:`.

## Alternativas evaluadas y descartadas

- **Insync** (USD $40 una vez, GUI tipo Drive oficial): descartado por costo y por preferencia CLI de Koichi.
- **google-drive-ocamlfuse**: válido pero menos features que rclone (sin bisync, sin multi-cloud).
- **GNOME Online Accounts**: solo navegación básica, no es sync real.
