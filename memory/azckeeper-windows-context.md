# AZCKeeper — Updater + builder (RESUELTO 2026-05-28)

Hasta 2026-05-28 el updater y el pipeline de build vivían fuera de git en `/home/kelsie/Documents/AZCKeeper/`. En esta fecha se restructuró el repo a layout sln-at-root y se incorporaron al control de versiones. El repo `KoichiFumikatsu/AZCKeeper` es ahora autosuficiente para reconstruir un release end-to-end.

Activadores de este archivo: **"AZCKeeper Updater"**, **"builder"**, **"build-release"**, **"install.bat"**, **"hallazgo E"** (ZIP sin firma), **"feature web blocking"**, **"DevLinux"**, **"Henao2007"**, **"desarrollo branch"**.

---

## Estado de ramas (2026-05-28)

| Rama | Estado (2026-06-03 post-consolidación) | Notas |
|---|---|---|
| `origin/master` | f8aafa3 | **ABANDONADO** (2026-06-09). NO es producción. Buildea updater roto (sin fix proxy `bdf8357`). No usar para builds. |
| `origin/DevLinux` | 70c97a6 | **Branch principal y DEFAULT del repo en GitHub** (2026-06-09). Fuente de verdad. Prod corre builds de aquí (release activo 3.0.2.2). Restructura + updater (incl. `bdf8357` proxy bypass) + builder + web-blocking inerte + install-coverage + build `-p:Version` (`70c97a6`). WIP sin commitear: killer/install (`azc-killer.ps1` + `install.bat`) |
| ~~`origin/feature/web-blocking`~~ | BORRADA 2026-06-03 | Sus 4 commits viven en DevLinux vía FF merge. Branch eliminada local y remoto |
| `origin/desarrollo` | 1613059 | Rama HUÉRFANA del aprendiz Henao2007. NO mergear (regresiones — ver abajo) |
| master local | 5072c2f | 3 commits adelante de origin/master (no pushed); apuntando al estado pre-consolidación |

---

## Layout del repo tras restructura

Tres commits locales en `master` del repo `/home/kelsie/AZCKeeper/` (pendientes de push a `master`, ya pusheados a `DevLinux`):

| Commit | Cambio |
|---|---|
| `a4b0efb` | `chore: restructure to sln-at-root layout` — `git mv` del cliente a subdir `AZCKeeper_Client/` |
| `932cfb1` | `feat: add AZCKeeperUpdater project` — `.csproj` + `Program.cs` |
| `5072c2f` | `feat: add Visual Studio solution and build pipeline` — `.sln`, `build-release.bat`, `install.bat` |

```
/home/kelsie/AZCKeeper/
  AZCKeeper.sln               (referencia ambos proyectos por path relativo)
  build-release.bat           (pipeline de build, requiere Windows)
  install.bat                 (instalador final, se empaqueta dentro del ZIP)
  .gitignore                  (cubre bin/, obj/, *.user, .env, *.sql, *.suo)
  AZCKeeper_Client/           (cliente C# .NET 8 WinForms)
    AZCKeeper_Client.csproj
    Auth/ Blocking/ Config/ Core/ Logging/ Network/
    Startup/ Tracking/ Update/ Web/
    Program.cs ...
  AZCKeeperUpdater/
    AZCKeeperUpdater.csproj
    Program.cs                (172 líneas)
```

El `AZCKeeper_Client.csproj` ya tenía un target `CopyUpdaterIfExists` con paths `..\AZCKeeperUpdater\...` que antes de la restructura apuntaban fuera del repo (warning silencioso). Tras la restructura resuelven correctamente al sibling.

---

## AZCKeeperUpdater — flujo y código

**Stack:** `net8.0-windows`, `WinExe`, `SelfContained=true`, `PublishSingleFile=true`, `PublishTrimmed=true`, `PublishReadyToRun=true`, `IncludeNativeLibrariesForSelfExtract=true`, RID `win-x64`. Sin NuGet refs.

**CLI:** `AZCKeeperUpdater.exe <targetDir> <sourceDir> <oldExe>` — los tres argumentos son obligatorios; si faltan, log + sleep 3s + exit.

**Flujo en `Program.cs`:**

1. Crea log en `%LOCALAPPDATA%\AZCKeeper\Logs\updater_yyyyMMdd_HHmmss.log`.
2. **Espera ciega `Thread.Sleep(3000)`** — no detecta el proceso cliente, solo asume que ya cerró. Riesgo: si el cliente tarda >3s en cerrar, la copia puede fallar por file lock.
3. Verifica que `sourceDir` y `targetDir` existen (sino, log + sleep + exit).
4. `Directory.GetFiles(sourceDir, "*.*", SearchOption.AllDirectories)` → para cada archivo: crea subdir destino si no existe, `File.Copy(file, targetPath, overwrite: true)`. Log por cada archivo con índice.
5. `Process.Start(oldExePath, UseShellExecute=true, WorkingDirectory=targetDir)` para relanzar el cliente. Si `oldExePath` no existe, log de warning pero continúa.
6. Sleep 2s.
7. **Autodestrucción:** crea `%TEMP%\cleanup_updater.bat` con `timeout /t 2 && del /f /q <updater.exe> && del /f /q <bat>`, lo lanza con `cmd.exe /c` (window oculta). Nota en el código: `.NET 8 + UseShellExecute=false + .bat` da `Win32Exception` directo, por eso se invoca `cmd.exe /c` explícitamente.

**Try-catch global** en `Main` captura excepciones y loguea stack trace.

**Invocación desde el cliente:** `AZCKeeper_Client/Update/UpdateManager.cs` → `DownloadAndInstallAsync()`:
- GET ZIP de GitHub Releases (`keeper_client_releases.download_url`) → `File.WriteAllBytesAsync` → `ZipFile.ExtractToDirectory(zipPath, extractPath)`.
- Verifica que `AZCKeeperUpdater.exe` esté en el ZIP extraído — si no, return (no actualiza).
- Construye args: `currentDir = Path.GetDirectoryName(Process.GetCurrentProcess().MainModule.FileName)`, llama `Process.Start(updaterPath, $"\"{currentDir}\" \"{extractPath}\" \"{currentExe}\"")`.
- `await Task.Delay(1000)` → `Application.Exit()`.

**Sin verificación de hash ni firma del ZIP** — hallazgo [E] sigue abierto. Ahora el updater está en el repo, así que el fix es atacable desde Linux.

---

## build-release.bat — pipeline de build (5 pasos)

**Hardcoded:** `VERSION=3.0.2.0`, `BUILD_DIR=%~dp0build`, `CONFIG=Release`, `RUNTIME=win-x64`. Bump `VERSION` in-file por release. (Posible mejora futura: parametrizar via arg `build-release.bat 3.0.2.1`.)

| Paso | Acción |
|---|---|
| [1/5] | `rmdir /s /q build/` + `mkdir build/`, `mkdir build/package/` |
| [2/5] | `cd AZCKeeperUpdater && dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:PublishTrimmed=true -o build/updater` |
| [3/5] | `cd AZCKeeper_Client && dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=false -p:PublishReadyToRun=true -p:PublishTrimmed=false -o build/package` |
| [4/5] | `copy build/updater/AZCKeeperUpdater.exe build/package/` + `copy install.bat build/package/` + `del build/package/*.pdb` |
| [5/5] | `powershell Compress-Archive -Path build/package/* -DestinationPath build/AZCKeeper_v{VERSION}.zip -Force` |

Imprime tamaño del ZIP en MB y bytes — el número de bytes va al campo `size_bytes` de `keeper_client_releases` en el panel admin `/admin/releases.php`.

**Requiere Windows** (powershell Compress-Archive, paths con `\`, `setlocal enabledelayedexpansion`). El cliente publica con multi-file + ReadyToRun (no trimmed) para mantener funcionalidad WinForms; el updater publica single-file trimmed (puede permitírselo porque es código mínimo sin reflection).

---

## install.bat — instalador empaquetado

Empaquetado dentro del ZIP por el paso [4/5] del builder. Flujo:

1. **Detección de usuario logueado:** `query user | findstr Activo` extrae el usuario con sesión activa. Esto es crítico porque el instalador puede ejecutarse como SYSTEM/DWService (canal remoto de IT) y `%LOCALAPPDATA%` apuntaría al perfil equivocado. Fallback: `%LOCALAPPDATA%` si no detecta.
2. `reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList" /s /v ProfileImagePath | findstr <user>` resuelve el path real del perfil.
3. `INSTALL_DIR=%USER_PROFILE%\AppData\Local\AZCKeeper\app`.
4. `taskkill /F /IM AZCKeeper_Client.exe` y `taskkill /F /IM AZCKeeperUpdater.exe`.
5. `rmdir /S /Q %INSTALL_DIR%` (instalación previa) → `mkdir %INSTALL_DIR%`.
6. `xcopy /Y /E /I "%~dp0*.*" "%INSTALL_DIR%"` (copia desde donde se extrajo el ZIP).
7. `start "" "%INSTALL_DIR%\AZCKeeper_Client.exe"`.

---

## Lo que NO se incorporó al repo (decisiones diferidas)

Estos siguen en `/home/kelsie/Documents/AZCKeeper/`:

- `reparar-cliente.bat` — hardcoded a `v3.0.0.11` (vieja). Pendiente actualizar/parametrizar antes de subir.
- `reparar-clienteOLD.bat` — legacy, no subir.
- `FallbackTest/` — proyecto .NET 9 que valida el fallback HTTPS→HTTP. Scope test, no subido (TargetFramework distinto al resto).
- `pipezafra_soporte_db_OnlyKeep.sql` — schema-only de tablas `keeper_*` (196 KB). `.gitignore` global `*.sql` lo bloquea; subirlo requiere excepción explícita.
- `pipezafra_soporte_db.sql` — dump completo (209 MB). Nunca al repo.
- `firma-email/` — firma corporativa AZC, fuera de scope Keeper.
- `memory/` local (Documents) — se migró su `decisions.md` a memoria global (ver `decisions.md` de memoryClaude-main).
- Binarios compilados `bin/`, `obj/`, `build/` — regenerables, cubiertos por `.gitignore`.
- `Web/.env` — credenciales producción (DB_PASS, APP_KEY, CRON_API_KEY). Correcto que esté fuera de git.

## Copia local desactualizada

`/home/kelsie/Documents/AZCKeeper/AZCKeeper_Client/` está ~2 meses atrás del repo (Mar 7 vs May 8). Decisión 2026-05-28: dejarla como está, solo retiene `.env`, dumps SQL y scripts no incorporados. **El repo es fuente de verdad** para el cliente, updater y builder.

## Próximos pasos pendientes

1. Decidir merge `DevLinux` → `master` (origin/master sigue en f8aafa3, DevLinux en f0dac1c). Sub-decisión: incluir web-blocking inerte o sacarlo del release (ver hallazgo [M] en `project_azckeeper.md`).
2. Atacar hallazgo [E]: verificación de hash/firma del ZIP en el updater (ya viable desde Linux con `build-release.sh`).
3. ~~Build desde Linux~~ — RESUELTO 2026-06-03: `build-release.sh` en `/home/kelsie/AZCKeeper/` cross-compila a Windows desde Fumilinux usando SDK Microsoft user-local (`/home/kelsie/.dotnet/`, NO el del apt Ubuntu que es subset). Output `build/AZCKeeper_v<version>.zip` (~75 MB) idéntico al `.bat`. Flag clave `-p:EnableWindowsTargeting=true`. El `.bat` se mantiene como backup.

---

## Revisión de `origin/desarrollo` (2026-05-28) — NO MERGEAR

Rama del aprendiz Henao2007 (`stivenhenaovargas89@gmail.com`). **Rama huérfana**: sin historia común con master. Inicializada por GitHub Copilot SWE Agent el 2026-04-08 (commits `0afb9c8` "Initial plan" + `693f5f6` "RESUMEN_PROYECTO.md"). Último commit `1613059` (2026-05-08) con mensaje "Proyecto completo, falataria instalar el sistema en cada pc con privilegios de admin" — **force-pushed** sobre estado anterior `e7a5039`.

### Feature válida (rescatada en `feature/web-blocking`)

Sistema de bloqueo web ~1100 líneas, autocontenido en `Blocking/`:
- `HostsFileBlocker.cs` — bloqueo por `C:\Windows\System32\drivers\etc\hosts`
- `LocalWebBlockProxy.cs` — proxy local HTTP/HTTPS
- `SystemProxyManager.cs` — gestiona registry de proxy del sistema (HKCU)
- `WebBlockingManager.cs` — orquestador con `Initialize(WebBlockingConfig, apiBaseUrl)`, `ApplyRemotePolicy(...)`, `Shutdown()`

### Regresiones críticas (NO importadas)

| # | Archivo | Regresión | Impacto |
|---|---|---|---|
| 1 | `Web/src/Db.php` | Check hardcoded `if ($db !== 'azckeeper_local') throw` | Mata backend en prod (DB real es `pipezafra_keep`) |
| 2 | `Web/src/Db.php` | Eliminado fallback `.env.backup` | Sin redundancia de BD |
| 3 | `Web/src/Endpoints/ForceHandshake.php` | **Parche [F] REVERTIDO** — auth admin cookie removida | Endpoint público sin auth |
| 4 | `Web/public/index.php` | **Parche [D] REVERTIDO** — `/client/device-lock/unlock` eliminada | Cliente recibe 404 al desbloquear |
| 5 | `Web/migrations/` | **15 archivos SQL borrados** | Imposible reconstruir BD desde cero |
| 6 | `Blocking/KeyBlocker.cs:553` | **Parche [C] REVERTIDO** — `PIN recibido='{pin}', PIN guardado='{_unlockPin}'` plaintext en log | Vulnerabilidad de logs reintroducida |
| 7 | `Blocking/KeyBlocker.cs` | `System.Timers.Timer` → `System.Windows.Forms.Timer` | Regresión documentada de fix anterior |
| 8 | `AZCKeeper_Client.csproj` | Referencia `scripts\ProxyTestHarness\**\*.cs` (carpeta no commiteada) | Build falla en clonación limpia |
| 9 | `AZCKeeper.sln` propio | Solo referencia `AZCKeeper_Client.csproj`, no `AZCKeeperUpdater` | Incompatible con sln-at-root |
| 10 | `Config/ConfigManager.cs` | `ApiBaseUrl = "http://localhost/AZCKeeper/Web/public/index.php/api/"` | URL dev local hardcoded |

Conclusión: **base obsoleta** (pre-parches [C][D][F] del 2026-05-08). Pendiente: conversación con Henao2007 para que su próximo trabajo parta del estado correcto (`master` o `DevLinux`, no de su rama huérfana).

---

## `feature/web-blocking` — cherry-pick selectivo (2026-05-28) + fix 2026-06-01

Branch creada desde `5072c2f` (DevLinux/master local). 4 commits sobre eso:

| Commit | Fecha | Cambios |
|---|---|---|
| `a6d69ec` | 2026-05-28 | `feat(blocking)`: 4 archivos nuevos en `AZCKeeper_Client/Blocking/` (1093 líneas, namespace `AZCKeeper_Cliente.Blocking`) |
| `f67d87e` | 2026-05-28 | `feat(client)`: ConfigManager (WebBlockingConfig class + property + init), ApiClient (EffectiveWebBlocking DTO), CoreService (field + Initialize en InitializeModules + Shutdown + bloque policy apply tras Blocking en PerformHandshake) — 65+/−1 |
| `7160d0a` | 2026-05-28 | `feat(web)`: PolicyRepo.getWebBlockedDomains() con cache 60s + normalizeWebBlockedDomain(), InputValidator.validateDomainArray(), ClientHandshake normaliza webBlocking, bootstrap.php (fix latente: requires de `ClientReEnroll.php` + `WindowEpisodeBatch.php` faltantes), policies.php (UI nueva "Web Blocking" + `syncGlobalWebBlockingDomains()` helper que escribe en `keeper_policy_assignments.policy_json.webBlocking` global tras save_leisure_apps) — 256+/−10 |
| `1df4130` | 2026-06-01 | `fix(web)`: habilita Web Blocking como módulo independiente en modal. `policies.php` — modal `enabled` y textarea de dominios pasan a editables (antes `enabled` se derivaba de `domains.length > 0`); "Por Ventana" vuelve a `window_title LIKE` para tracking de ocio (Henao2007 lo había repurposeado como entrada de dominios); `save_leisure_apps` deja de llamar `syncGlobalWebBlockingDomains()` y la función se elimina. `PolicyRepo.php` — `getWebBlockedDomains()` + `normalizeWebBlockedDomain()` + caches eliminadas (código muerto sin caller tras quitar la sync). Net +8/-130 líneas en 2 archivos. |

### Flujo end-to-end de la feature

1. Admin configura dominios en panel Policies → sección "Por Ventana" (renombrada para web). Save dispara `syncGlobalWebBlockingDomains($pdo, $wins)` → UPSERT en `keeper_policy_assignments` scope=global con `version++`.
2. Cliente hace handshake → backend lee policy → `ClientHandshake.php` normaliza `effective.webBlocking` (enabled bool, syncIntervalSeconds floor 300, domains via `InputValidator::validateDomainArray`).
3. Cliente C# `CoreService.PerformHandshake` recibe `effective.WebBlocking` → escribe `_configManager.CurrentConfig.WebBlocking` → llama `_webBlockingManager.ApplyRemotePolicy(webBlocking, version, apiBaseUrl)`.
4. `WebBlockingManager` aplica: escribe bloque en hosts file (tag AZCKeeper), levanta `LocalWebBlockProxy` en puerto local, configura `SystemProxyManager` en HKCU Internet Settings.

### Cambios NO importados de origin/desarrollo (scope separado o regresiones)

- `Db.php`, `ForceHandshake.php`, `public/index.php`, `KeyBlocker.cs`, `AZCKeeper.sln`, migrations — todas regresiones.
- `Update/UpdateManager.cs` (+119), `Network/ApiClient.cs` resto (+12), `Program.cs` (+27), `Config/ConfigManager.cs` (NormalizeLegacyTimerValues, default HandshakeIntervalSeconds 60→300, campos para rate-limit del UpdateManager) — scope separado, no revisados.

---

## Paquete para Windows (2026-05-28) — ENVÍO PENDIENTE

Creado en `/home/kelsie/Documents/`:
- `AZCKeeper-feature-web-blocking-2026-05-28.zip` (390 KB, 144 archivos) — `git archive feature/web-blocking` con exclusión explícita `:(exclude).vs` `:(exclude)*.user`.
- `AZCKeeper-feature-web-blocking-2026-05-28-INSTRUCCIONES.md` (8.9 KB) — setup `.env` (DEV recomendado primero: `/home/kelsie/Downloads/env.dev`), `build-release.bat`, plan de prueba en 4 fases (smoke sin DB → política en admin → validación cliente con hosts/registry/netstat/navegador → desbloqueo), paths de logs, captura HTTP (Fiddler/Wireshark), habilitar OpenSSH Server en Windows (PowerShell admin: `Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0`, firewall rule LocalPort 22, fix `administrators_authorized_keys` si cuenta admin), setup llave `~/.ssh/azckeeper_winportable` desde Fumilinux, comandos PowerShell vía SSH (tail logs, ver proxy, hosts, reiniciar cliente).

**Estado:** paquete preparado en Fumilinux. **No transferido al portátil Windows aún.** No probado.

### Hallazgos latentes detectados en este flujo

1. **`.vs/` tracked en el repo** — el `.gitignore` no cubre `.vs/`. Pendiente: agregar regla + `git rm --cached -r .vs/`.
2. **`AZCKeeper_Client.csproj.user` tracked al root** — legacy desde antes del gitignore `*.user`. Pendiente: `git rm --cached AZCKeeper_Client.csproj.user`. (Workaround: en el archivo de paquete se excluyó con `:(exclude)*.user`.)
3. **Backend PHP sin autoloader** — no hay `composer.json`. Las clases `Keeper\Endpoints\*` se cargan por `require_once` individual en `bootstrap.php`. `WindowEpisodeBatch.php` y `ClientReEnroll.php` estaban referenciadas en routes de `public/index.php` pero faltaban en bootstrap (latent bug). El commit `7160d0a` de feature/web-blocking lo arregla. En master sigue roto si esas rutas se golpean.

---

## Pendientes ordenados por prioridad

1. **Transferir el paquete al portátil Windows** — USB / SCP / pegado manual. NO HECHO.
2. Setup `.env` en Windows (DEV primero).
3. Build `build-release.bat` en Windows → verificar ZIP `build\AZCKeeper_v3.0.2.0.zip`.
4. Habilitar OpenSSH Server en el portátil + setup llave desde Fumilinux para tratamiento remoto.
5. Smoke test del web blocking contra DEV (todas las 4 fases del instructivo).
6. Si DEV OK → repetir contra PROD.
7. Conversación con Henao2007 sobre la base obsoleta que usó.
8. Decidir cuándo merge `feature/web-blocking` y/o `DevLinux` a `master` y push de master.
9. Higiene del repo: agregar `.vs/` a `.gitignore`, `git rm --cached -r .vs/`, `git rm --cached AZCKeeper_Client.csproj.user`. Commit aparte.
10. Atacar hallazgo [E] (firma/hash del ZIP en updater) — ahora atacable desde el repo.
