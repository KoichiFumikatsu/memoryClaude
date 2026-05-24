# Seguridad Fumilinux — escaneo de descargas

Implementado 2026-05-24 para reducir riesgo de malware en builds de juegos descargados (workflow TL Games). Solo se implementó **Capa 1**. Capas 2-4 decididas no instalar.

## Estado del sistema

| Componente | Estado |
|---|---|
| UFW | inactive |
| AppArmor | active, 114 perfiles enforce |
| ClamAV | 1.4.4 (engine + daemon + freshclam) ACTIVO |
| vt-cli | 1.3.1 en `/usr/local/bin/vt` |
| Wrapper `scan` | `~/.local/bin/scan` (Python) |
| sudoers | `kelsie ALL=(ALL) NOPASSWD: ALL` — riesgo aceptado por Koichi |
| firejail / opensnitch / rkhunter / lynis | NO instalados (decisión Koichi) |

## ClamAV

- Daemon: `clamav-daemon.service` (enabled + active). Socket: `/var/run/clamav/clamd.ctl`. Corre como user `clamav`.
- Auto-update firmas: `clamav-freshclam.service` (4h aprox). Bases: daily.cvd, main.cvd, bytecode.cvd.
- Escaneo via socket: `clamdscan --fdpass <file>` (rápido, daemon-based). Fallback CLI: `clamscan`.
- Trampa: en primera instalación el daemon arranca antes de que freshclam baje firmas. Fix: detener freshclam, correr `sudo freshclam` manual, arrancar daemon.

## VirusTotal CLI

- Binario: `/usr/local/bin/vt` (descargado de https://github.com/VirusTotal/vt-cli/releases). NO hay paquete apt.
- Config: `~/.vt.toml` (perms 600). Contiene API key.
- Tier free: 4 req/min, 500/día, 15.5k/mes. Upload max 32MB.
- Comandos clave:
  - `vt file <sha256>` — info por hash (no consume upload quota, solo lookup)
  - `vt scan file <path>` — sube archivo (consume quota, escaneo nuevo)
  - `vt url <url>` — info de URL
  - `vt domain <domain>` / `vt ip <ip>`
- Nota: `vt info` NO existe. Comando real es `vt file <hash>`.

## Wrapper `scan` (~/.local/bin/scan)

```bash
scan <file_or_dir> [...]              # ambos motores
scan --no-vt <file>                   # solo ClamAV (sin quota)
scan --vt-delay 16 <file>             # delay entre requests VT (default 16s = ~4/min)
```

Salida: tabla por archivo con CLAMAV y VT (detected/total). Exit code 1 si hay detecciones.

## Lanzadores tipo .bat

Scripts:
- `~/.local/bin/scan-downloads` — escanea ~/Downloads (ClamAV + VT). Log + pausa al final.
- `~/.local/bin/scan-home` — escanea ~/ completo (solo ClamAV). Excluye caches, GoogleDrive, snap, browser caches.

Logs: `~/.cache/scan/<target>-<timestamp>.log`

.desktop launchers en `~/.local/share/applications/` (menú de apps via Super → "Escanear") y copias en `~/Desktop/` con `metadata::trusted=true` para doble-click.

**Trampa Ubuntu 24.04 GNOME**: el escritorio NO muestra íconos por default. Habilitar "Desktop Icons NG (DING)" en Settings → Extensions. Sin extensión, funcionan solo desde el menú de apps.

Terminal: `scan-downloads` o `scan-home`.

Estados VT:
- `N/M` — detected/total engines
- `unknown` — archivo nunca visto por VT (NO significa limpio — nadie lo subió)
- `err: ...` — fallo de API o red

Para subir un `unknown` (consume quota): `vt scan file <path>`.

## Decisiones tomadas (no re-debatir)

- **NOPASSWD sudo se queda**: Koichi conoce el riesgo, lo asume. No insistir.
- **Capa 2-4 no instaladas**: firejail (sandboxing), opensnitch (firewall per-app), rkhunter/lynis (audit) — decididas NO instalar en 2026-05-24. Si Koichi pide reforzar más adelante, reabrir.

## Comandos de mantenimiento

```bash
sudo systemctl status clamav-daemon clamav-freshclam
sudo systemctl stop clamav-freshclam && sudo freshclam && sudo systemctl start clamav-freshclam
$EDITOR ~/.vt.toml
curl -s https://api.github.com/repos/VirusTotal/vt-cli/releases/latest | grep browser_download_url | grep Linux64
```

## Workflow recomendado para descargas TL Games

1. Bajar el archivo a `~/Downloads/` (NO ejecutar todavía)
2. `scan ~/Downloads/<archivo>` → revisar ambos motores
3. Si VT dice `unknown` y el origen es sketchy: `vt scan file <archivo>` y esperar
4. Si ClamAV o VT detectan algo: borrar, NO ejecutar
5. Solo extraer/ejecutar si ambos motores limpios

Limitaciones:
- `unknown` en VT no implica limpio. Para builds indie nuevos es lo normal.
- ClamAV detecta malware conocido. Malware nuevo/targeted no aparece.
- Sin runtime defense (firejail/opensnitch). Si se ejecuta algo malicioso, corre con permisos completos + NOPASSWD sudo.
