# Strinova en Linux vía Steam (funcionando 2026-05-29)

Strinova (Steam appid `1282270`) usa **Anti-Cheat Expert (ACE)** a nivel kernel — no funciona con Proton estándar ni GE-Proton. Sí funciona con **dwproton** (fork de Proton-CachyOS con parches para juegos anime/gacha).

Verificado en Fumilinux (i7-1355U + Intel Iris Xe). Para torre casa (Ryzen 5 5500 + RX 6500 XT) los pasos deberían ser idénticos, con la salvedad del fix de benchmark en GPU NVIDIA (no aplica a AMD/Intel).

## Pasos exactos

### 1. Descargar dwproton

Releases: `https://github.com/dawn-winery/dwproton-mirror/releases`

Versión usada y confirmada: **dwproton-10.0-22-x86_64**. Versiones posteriores deberían servir igual mientras dawn-winery siga publicando.

```bash
# Obtener URL de la última release vía API
curl -s "https://api.github.com/repos/dawn-winery/dwproton-mirror/releases/latest" \
  | grep -E '"browser_download_url".*tar\.xz' \
  | cut -d '"' -f 4
```

### 2. Instalar en el path CORRECTO

**Crítico:** Steam lee de `~/.local/share/Steam/compatibilitytools.d/`, no de `~/.steam/steam/compatibilitytools.d/`. Si lo pones en el segundo, no aparece en el dropdown.

```bash
mkdir -p ~/.local/share/Steam/compatibilitytools.d
tar -xf dwproton-10.0-22-x86_64.tar.xz -C ~/.local/share/Steam/compatibilitytools.d/
```

### 3. Reiniciar Steam completamente

Steam solo escanea `compatibilitytools.d/` al arrancar. Si dwproton no aparece en el dropdown, no reiniciaste — `pkill steam` y abrir de nuevo.

### 4. Configurar el juego en Steam

- Clic derecho en Strinova → **Propiedades** → **Compatibilidad**
- Activar **"Forzar el uso de una herramienta de compatibilidad específica"**
- Seleccionar `dwproton-10.0-22-x86_64`

### 5. Launch options para audio (PipeWire)

En **Propiedades → Opciones de lanzamiento**:
```
PROTON_USE_WINEALSA=1 %command%
```

Sin esto el launcher crashea por audio en sistemas con PipeWire.

## Comportamiento esperado

- El launcher puede colgarse en "Launch..." la primera vez — cerrar y reabrir.
- También puede tirar `LowLevelFatalError` la primera vez, mismo fix (reintentar).
- Una vez dentro del menú, todo estable.

## Fix adicional solo para NVIDIA (no aplica a Intel iGPU ni AMD según guías)

Si faltan modelos en menú de agentes o las skins/armas se ven negras/oscuras:

Editar `GameUserSettings.ini` en:
```
~/.local/share/Steam/steamapps/compatdata/1282270/pfx/drive_c/users/steamuser/AppData/Local/Strinova/Saved/Config/WindowsNoEditor/GameUserSettings.ini
```

Cambiar `LastCPUBenchmarkResult` y `LastGPUBenchmarkResult` (vienen en negativo, lo que rompe el preset gráfico):
- `0` = potato
- `100` = low
- `200` = medium
- `500` = high (recomendado)
- `1500` = ultra

El número de benchmark sobrescribe todos los settings individuales, así que cambiarlo desde el menú in-game no sirve.

## Por qué dwproton y no GE-Proton

GE-Proton tiene parches generalistas pero no incluye los hacks específicos que ACE necesita para no detectar el entorno como sospechoso. dwproton es downstream de Proton-CachyOS + parches de dawn-winery enfocados en anime/gacha (Wuthering Waves, Genshin, ZZZ, Strinova, etc.).

## Riesgo a futuro

Cada update de Strinova puede romper el anti-cheat. Si deja de funcionar tras un update, revisar releases nuevas de dwproton — suelen sacar parche en pocos días.
