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

**[2026-05-15] Compañero — stack Flutter** — Flutter (Dart) sobre React Native para el juego móvil Compañero.
*Why: mejor soporte de animaciones 2D, una sola codebase iOS+Android, y el pipeline de arte (sprites pre-renderizados) encaja mejor con Flutter que con RN.*

**[2026-05-15] Compañero — arte como sprites pre-renderizados** — Mar renderiza cada estado/expresión/animación del Blob en Blender y exporta sprite sheets PNG con alpha. No se usa un motor 3D en el cliente.
*Why: mantiene la fidelidad visual del clay render 3D sin cambiar el stack del cliente ni requerir Blender en runtime.*

**[2026-05-15] Compañero — Hábitos arte ≠ Mecánica en v1 (Opción A, pendiente confirmación)** — Los mundos visuales se llaman "Sueño+Conexión" e "Hidratación+Nutrición" pero los stats internos son solo Sueño e Hidratación. Nutrición y Conexión son flavor narrativo en v1 sin sistema propio.
*Why: no hay mecanismo de verificación viable para Nutrición (comida) ni Conexión (social) en v1. Se implementan como stats reales en v2. Pendiente aprobación del dueño del producto.*

**[2026-05-15] Compañero — HealthKit/Health Connect diferido a v2** — La integración con health (pub.dev) se deja para v2. En v1 el movimiento se verifica por acelerómetro (sensors_plus), el sueño por proxy (pantalla apagada + teléfono estático), el agua por honor system.
*Why: simplifica el MVP y evita el proceso de aprobación de Health API en tiendas para v1.*

**[2026-05-16] KelsieApp — ConfirmSheet como patrón estándar de destructivos** — Todos los `confirm()` nativos del browser fueron eliminados de KelsieApp. El componente `components/ui/ConfirmSheet.tsx` es el único patrón válido para confirmaciones de eliminación. Se exporta desde `components/ui/index.ts`.
*Why: `confirm()` bloquea el hilo, rompe el design system y no funciona en ciertos contextos de WebView/Vercel. ConfirmSheet usa el BottomSheet existente con props `label`, `onConfirm`, `onClose`, `confirmLabel?`, `title?`.*

**[2026-05-16] KelsieApp — Touch targets mínimos 44×44px globales** — Regla no negociable en todos los módulos. Icon-only buttons usan `min-w-[44px] min-h-[44px] flex items-center justify-center` (Tailwind) o `{ minWidth: 44, minHeight: 44, display: 'flex', alignItems: 'center', justifyContent: 'center' }` (inline style). FABs `h-12 w-12` = 48px ya cumplen.
*Why: usuarios finales con TDAH y espectro autista. WCAG 2.5.5 Level AA.*

**[2026-05-16] KelsieApp — aria-label en icon-only buttons (no title)** — El atributo `title` no es leído consistentemente por screen readers en mobile. Todos los botones con solo un ícono deben usar `aria-label` descriptivo en español.
*Why: `title` fue encontrado en todos los módulos; reemplazado sistemáticamente.*
