# AZCKeeper — Contexto Windows pendiente de documentar

Claves de activación: **"AZCKeeper Updater"**, **"builder"**.

Cuando Koichi mencione estas claves, hay contexto adicional sobre AZCKeeper que NO está en:
- Repo local `/home/kelsie/AZCKeeper/`
- GitHub `KoichiFumikatsu/AZCKeeper`
- Memorias actuales

Este contexto solo existe en la máquina Windows donde se hace el build del cliente C# .NET 8.

## Qué falta levantar (al primer acceso al portátil de pruebas)

- **AZCKeeperUpdater** — proyecto/binario del updater (se lanza desde el cliente principal para reemplazar el ejecutable durante auto-actualización). El cliente espera que `AZCKeeperUpdater.exe` esté dentro del ZIP descargado de GitHub Releases, pero su código fuente no aparece en el repo principal.
- **Builder** — flujo, scripts y configuración del proceso de build de Release que solo existe en Windows.
- Probables artefactos adicionales: certificados de firma, claves locales, paths de salida hardcoded, scripts de empaquetado del ZIP que termina en `keeper_client_releases.download_url`, configuración de `dotnet publish` con flags específicos.

## Procedimiento al acceder al portátil

1. Inventario: qué proyectos .sln/.csproj existen, en qué carpetas, qué versión de .NET SDK, qué herramientas (Visual Studio, Rider, dotnet CLI standalone).
2. Levantar código fuente de AZCKeeperUpdater y del builder a un repo (sea `KoichiFumikatsu/AZCKeeper` como subcarpeta o un repo separado).
3. Documentar el flujo end-to-end: dónde se compila, qué se firma, cómo se empaqueta, cómo se sube el ZIP, cómo se actualiza `keeper_client_releases`.
4. Persistir hallazgos en memoria dual (auto-memory + memoryClaude-main) y en CLAUDE.md del repo AZCKeeper.

## Meta a futuro

Una vez documentado y subido a repo, el objetivo es poder hacer el build completo (cliente + updater + empaquetado + release) desde Fumilinux con `dotnet publish -r win-x64 -c Release --self-contained`, eliminando la dependencia del escritorio Windows. Hoy esto es bloqueador para [C] (deploy del PIN fuera de logs).
