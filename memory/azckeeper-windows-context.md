# AZCKeeper — Updater + builder (RESUELTO 2026-05-28)

Hasta 2026-05-28 el updater y el pipeline de build vivían fuera de git en `/home/kelsie/Documents/AZCKeeper/`. En esta fecha se restructuró el repo a layout sln-at-root y se incorporaron al control de versiones. El repo `KoichiFumikatsu/AZCKeeper` es ahora autosuficiente para reconstruir un release end-to-end.

Activadores de este archivo: **"AZCKeeper Updater"**, **"builder"**, **"build-release"**, **"install.bat"**, **"hallazgo E"** (ZIP sin firma).

---

## Layout del repo tras restructura

Tres commits locales en `master` del repo `/home/kelsie/AZCKeeper/` (pendientes de push):

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

1. Push de los tres commits (`a4b0efb`, `932cfb1`, `5072c2f`) a `origin/master`.
2. Verificación end-to-end de `build-release.bat` desde el portátil Windows tras clonar repo limpio.
3. Atacar hallazgo [E]: verificación de hash/firma del ZIP en el updater (ya viable desde Linux).
4. Meta a futuro: portar `build-release.bat` a un script equivalente que corra en Fumilinux con `dotnet publish` puro + `zip` GNU — eliminando dependencia del portátil Windows para builds.
