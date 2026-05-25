# Suwayomi — Lector de manga en Linux

Equivalente a Mihon/Tachiyomi para Linux. Instalado 2026-05-25.

## Componentes

- **Suwayomi-Server v2.2.2100** JAR en `~/apps/suwayomi/`
- **Suwayomi-JUI v1.3.3** instalado vía .deb en `/opt/suwayomi-jui/`
- **Java 17** (`openjdk-17-jre-headless`)

## Arranque automático

Servicio systemd de usuario: `~/.config/systemd/user/suwayomi-server.service`
- Arranca al login, reinicia si falla
- Puerto: `localhost:4567`

```bash
systemctl --user status suwayomi-server
systemctl --user restart suwayomi-server
```

## Lanzador

- Script: `~/apps/suwayomi/suwayomi-launch.sh` — espera el puerto y abre la UI
- `.desktop` GNOME: `~/.local/share/applications/suwayomi.desktop`

## Extensiones

Agregar fuentes de manga en **Settings > Extensions** dentro de la app (mismas que Mihon).
