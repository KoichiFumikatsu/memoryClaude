# Suwayomi — Lector de manga en Linux

Equivalente a Mihon/Tachiyomi para Linux. Instalado 2026-05-25.

## Componentes

- **Suwayomi-Server v2.2.2100** JAR en `~/apps/suwayomi/`
- **Suwayomi-JUI v1.3.3** instalado vía .deb en `/opt/suwayomi-jui/`
- **Java 21** (`openjdk-21-jre-headless`) — el JAR es class file v65.0; Java 17 falla con UnsupportedClassVersionError

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

## Importar backup de Mihon

El `.tachibk` de Mihon moderno es gzip válido. Renombrarlo a `.proto.gz` y usar GraphQL multipart — NO el endpoint REST `/api/v1/backup/import` (no restaura correctamente):

```bash
curl -s "http://localhost:4567/api/graphql" \
  -F 'operations={"query":"mutation($backup: Upload!) { restoreBackup(input: { backup: $backup }) { id status { mangaProgress totalManga state } } }", "variables": {"backup": null}}' \
  -F 'map={"0": ["variables.backup"]}' \
  -F '0=@/ruta/backup.proto.gz'
```

Librería restaurada 2026-05-25: 281 manga, categorías Fav. M / Mangas / Fav. D / Doujins.
