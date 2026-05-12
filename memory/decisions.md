# Decisions Log

## Format
Each entry: **[Date] Topic** — decision made, and *why*.

---

## Decisions

**[2026-05-07] Sistema de memoria** — sistema de archivos en `/memory/` dentro del repo `memoryClaude`, con commit/push automático al final de cada sesión vía stop hook.
*Why: más fácil de diff y actualizar por archivo sin tocar la config principal. Portable entre máquinas vía GitHub.*

**[2026-05-07] Ruta de carrera** — desarrollo del perfil de liderazgo técnico, no pivot a IC remoto.
*Why: Koichi ya tiene una posición de liderazgo real con 4 años de historial; ese perfil es más competitivo a mediano plazo que empezar IC desde cero.*

**[2026-05-07] Prioridad de desarrollo profesional** — inglés B2 es la prioridad única e irremplazable; todo lo demás (certs, GitHub, LinkedIn) es secundario mientras no llegue a B2.
*Why: es el único bloqueador duro para el mercado remoto internacional.*

**[2026-05-07] CVs generados** — dos versiones: `CV_Liderazgo_KoichiFumikatsu.docx` y `CV_IC_KoichiFumikatsu.docx`.
*Pendientes: subir Localfy a GitHub, actualizar repos, actualizar LinkedIn, generar PDF cuando el contenido esté aprobado.*

**[2026-05-07] Descripción de KelsieApp en entrevistas** — "arquitectura diseñada y dirección de implementación con asistencia de IA". No presentar como desarrollo propio completo.
*Why: la implementación fue ejecutada con IA generativa; presentarlo de otro modo es impreciso y puede ser problemático en entrevistas técnicas.*

**[2026-05-07] Stack de memoria en Linux** — repo en `/home/kelsie/memoryClaude-main/`, remote SSH `git@github.com:KoichiFumikatsu/memoryClaude.git`. Stop hook adaptado de ruta Windows a Linux.
*Why: máquina de trabajo es Ubuntu, no Windows.*

**[2026-05-07] Delegación operativa — metodología acordada** — Koichi lista tareas donde es el único que sabe resolverlas → se convierten en documentación estructurada → se asigna formalmente a alguien del equipo.
*Estado: pendiente de iniciar. Koichi debe entregar lista inicial.*

**[2026-05-08] AZCKeeper — regla de prueba de escritorio** — cada parche aplicado a AZCKeeper debe tener prueba de escritorio que confirme flujo, datos enviados y recibidos. No dar un hallazgo por cerrado sin esa verificación.
*Why: aplicación en producción con 200+ usuarios sin acceso directo a equipos.*

**[2026-05-08] AZCKeeper — keeper_device_locks** — tabla huérfana. El lock real es por policy_json (blocking.enableDeviceLock). No usar keeper_device_locks como referencia de estado de bloqueo.
*Why: 0 filas, ningún código la lee ni escribe actualmente.*

**[2026-05-08] AZCKeeper — force-handshake** — el endpoint POST /client/force-handshake NO es llamado por el panel. El panel tiene su propia SQL inline en policies.php:123. El endpoint estaba expuesto sin auth y es código muerto desde el panel.
*Why: confusión entre ruta API y funcionalidad del panel.*

**[2026-05-12] Gaming en Linux — GE-Proton resuelve juegos japoneses** — Umamusume Pretty Derby (Steam) en Ubuntu 24.04 GNOME+Xorg funciona con GE-Proton10-34. Proton Experimental deja pantalla en blanco. Steam instalado como snap en `/home/kelsie/snap/steam/`.
*Why: Proton Experimental no renderiza el downloader de Cygames; GE-Proton tiene parches específicos para juegos japoneses.*

**[2026-05-12] Wallpapers animados en GNOME+Xorg — sin solución limpia** — linux-wallpaperengine compilado en `/home/kelsie/linux-wallpaperengine/`. GNOME ignora X root window; xwinwrap no funciona con GNOME porque Mutter dibuja encima. Hanabi (extensión GNOME) no quedó instalada correctamente. Wallpaper engine de Steam corre vía Proton y no puede afectar el escritorio Linux.
*Why: GNOME+Xorg no tiene soporte nativo para wallpapers animados sin extensiones específicas.*
