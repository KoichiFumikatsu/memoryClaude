# Decisions Log

## Format
Each entry: **[Date] Topic** — decision made, and *why*.

---

## Decisions

**[2026-05-28] Hardware swap Casa↔Oficina — CPU + RAM** — Intercambio físico de CPU y RAM entre Torre Casa y Torre Oficina, sin gasto. Oficina queda con Ryzen 5 5500 (Zen 3, 6c/12t) + 24GB RAM (eran de Casa). Casa queda con Ryzen 5 2400G + 8GB DDR4-2400 (eran de Oficina). GPUs y PSUs no se mueven. Placa Oficina (Gigabyte B450M DS3H V2) tiene BIOS F63 que ya soporta Zen 3 — no requiere update. RX 570 queda en Oficina obligatorio porque 5500 no tiene iGPU. Detalle completo en `hardware-3-torres.md`.
*Why: upgrade más grande gratis posible. Oficina es donde Koichi hace AI gen (sd-server + Ollama + Open WebUI), saturaba 8GB RAM con swap constante. 24GB elimina ese cuello. Casa solo se usa para gaming, no le molesta el downgrade. Cuello restante será GPU 4GB (RX 570) — futuro upgrade a RX 6700 XT o RTX 3060 12GB ~$200-280 cuando haya presupuesto.*

**[2026-05-28] AI Image — RX 6500 XT descartada como upgrade** — La RX 6500 XT (que Koichi tiene en Casa y AZC) es **downgrade efectivo** vs RX 570 para Stable Diffusion. Misma VRAM 4GB pero bus 64-bit vs 256-bit + PCIe 4.0 ×4 vs 3.0 ×16. Cuando hay swap a RAM (caso permanente con Pony 6.5GB en 4GB VRAM), la 6500 XT es 2-4× más lenta. Diseñada para gaming 1080p donde todo cabe en VRAM, no para AI.
*Why: AMD vendió la 6500 XT con specs recortadas durante GPU shortage 2022. Para gaming pasa, para workloads memory-bound como SD es desastre. RX 570 vieja > RX 6500 XT nueva en este caso.*

**[2026-05-28] AI Image — Protocolo batch nocturno obligatorio** — Cuando Koichi pida generación batch desatendida ("durante la noche", "mientras duermo", "mientras estoy fuera", "genera X seeds", "lo que alcances"), aplicar chain bash + nohup automáticamente sin preguntar. NO usar run_in_background individual por seed — crea cuello notificación↔respuesta que con usuario ausente 4x el tiempo total. Detalle en `~/projects/ia-gen/prompts/batch-generation-protocol.md` y memoria del proyecto.
*Why: el 2026-05-28 una sesión planeada de 6 seeds (1h12min) terminó tomando 4+ horas porque cada notificación esperaba que Koichi volviera. Koichi marcó esto explícitamente como problema: "no lo hiciste de manera autónoma, desperdiciaste 2 horas".*

**[2026-05-25] Claude Code launchers GNOME — 5 lanzadores .desktop por proyecto** — Creados en `~/.local/share/applications/claude-*.desktop`: TL Games (acceptEdits), Compañero (plan), Voice Assistant (default), AZCKeeper (plan), KelsieApp (plan). Todos con Opus 4.7. Patrón Exec usa `gnome-terminal --working-directory + bash -c "claude --model claude-opus-4-7 --permission-mode <mode> --name '<Proyecto>'; exec bash"`. Trampa documentada: `/init` NO carga contexto (lo crea); el contexto se carga automáticamente por cwd. Detalle completo en `claude-launchers-fumilinux.md`.
*Why: abrir Claude preconfig con un click en lugar de cd+flags manuales. Modo por proyecto: producción crítica (AZCKeeper/KelsieApp/Compañero) en plan para forzar planeación; TL Games en acceptEdits por iteración alta sobre scripts; Voice Assistant default por estar en producción pero no crítico.*

**[2026-05-25] AZCKeeper — testing del cliente C# vía RDP/SSH a portátil descartado** — Pruebas del cliente WinForms se hacen desde Fumilinux conectándose vía RDP (GUI) + SSH (CLI) a un portátil Windows reutilizado de los descartados de AZC. NO se usa VM (preferencia por hardware real) NO se usa Wine (SetWinEventHook + DPAPI + KeyBlocker no son representativos bajo Wine). Cliente RDP: xfreerdp/Remmina. Nota: Koichi indica que el setup RDP/SSH "a mano no le suele funcionar" — explicar siempre el procedimiento completo cuando se use.
*Why: Fumilinux no corre Windows y construir todo en VM es ciclo lento + falsos negativos en KeyBlocker/stealth. Los portátiles descartados existen, son gratis, y dan banco de pruebas físico equivalente al de un usuario final.*

**[2026-05-24] AI image local — REMOVIDO de Fumilinux, migrar a Torre 1** — `/home/kelsie/projects/ai-image/` borrado (36 GB liberados). El stack txt2img básico SD 1.5 funcionaba pero hires fix, detailer, img2img e inpaint disparan bug FakeTensor consistente en OpenVINO FX backend. Sin posibilidad de refinement → la iGPU Iris Xe llegó al techo para este caso de uso. Aprendizajes (trampas, URLs, prompts probados, settings) preservados en `ai-image-local.md` para configuración futura en Torre 1 (RX 570 con DirectML/ZLUDA en Windows o ROCm en Linux con `HSA_OVERRIDE_GFX_VERSION=9.0.0`).
*Why: estabilidad multi-stage (hires, img2img) es requisito para uso real; el bug FakeTensor no se puede patchear quirúrgicamente — aparece en cada path nuevo. GPU dedicada elimina el problema completo.*

**[2026-05-23] AI image local — SD.Next + OpenVINO operativo en Fumilinux** — Setup txt2img básico funcional: branch `dev` + patch en `modules/sd_models.py:848` (`low_cpu_mem_usage: not use_openvino`). Backend OpenVINO FX targeting Intel Iris Xe iGPU. SD 1.5 únicamente; SDXL no entra (OOM). Removido 2026-05-24 (ver entry de esa fecha).
*Why: combo torch 2.11+cpu/OpenVINO 2026.1 que SD.Next master pinea tiene bug FakeTensor en el path de compilación FX. El patch fuerza materialización eager para txt2img básico. Costo: ~110s primera generación SD 1.5.*

**[2026-05-22] TL Games — QA migrado a Groq** — `qa_renpy.py` ahora dispatcher `auto|groq|ollama`. Groq `llama-3.1-8b-instant` por default (free tier, 270x más rápido que Ollama CPU). Detalle en `tl-refactor-5-stages.md`.
*Why: Ollama llama3.2:3b saturaba CPU local; lint_qa stage timeout consistente. Groq free tier 14400 req/día > suficiente para uso personal.*

**[2026-05-22] TL Games — auto-diagnose post-job** — `pipeline_server.py` invoca `tools/diagnose.sh <job_id> --no-color` al finalizar cada job (toggle `diagnose.run_after_job` en settings). El reporte se guarda en `job.diagnose_report` y se muestra en dashboard en sección colapsable.
*Why: visibilidad inmediata del estado completo del sistema cuando algo termina mal, sin tener que ejecutar el comando manualmente. Best-effort: si falla, no rompe el pipeline.*

**[2026-05-22] TL Games — chmod +x automatico en copy_to_games_tl** — `_fix_executable_perms()` aplica +x a binarios Ren'Py (.sh, lib/py3-linux-*/<nombre>) y Unity (.x86_64) en destino Y origen tras la copia. Bug detectado con girlfriends_in_outer_worlds v0.3 cuyo binario interno venia sin bits ejecutables del ZIP original.
*Why: zips Linux mal empaquetados (descargados de itch.io/etc) suelen perder bits +x; rsync los preserva. Sin este fix el juego queda intraducible-pero-no-ejecutable tras el pipeline.*

**[2026-05-22] TL Games — auto-inyectar _force_<lang>.rpy si Ren'Py sin selector** — `_renpy_has_language_selector` detecta si el juego expone `change_language`/`_preferences.language` en su UI. Si no, `_renpy_inject_language_force` crea `game/_force_<lang>.rpy` con `config.default_language = "<lang>"`. Setting `renpy.force_language_if_no_selector: true`.
*Why: muchos juegos VN Ren'Py son monoglotas EN sin selector en menu; la traduccion existe pero no hay forma de activarla. El force.rpy es no destructivo y reversible (borrar archivo + persistent). Trampa: persistent existente del usuario sobrescribe el default — debe borrarse manualmente la primera vez.*

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

**[2026-05-19] n8n — Docker, no npm** — n8n se instala vía Docker (`docker.n8n.io/n8nio/n8n`, `--network=host`). Nunca intentar `npm install -g n8n` en esta máquina.
*Why: `isolated-vm` (dependencia nativa de n8n) no compila contra Node 18.19.1 por incompatibilidad de API V8. Falla con make exit code 2.*

**[2026-05-19] Ollama MCP — scope usuario** — El MCP `ollama-mcp` está registrado a nivel usuario (`claude mcp add --scope user`), no a nivel proyecto. Queda disponible en todas las sesiones de Claude Code sin configuración adicional.
*Why: la automatización con LLMs locales aplica a todos los proyectos (TL Games, AZCKeeper, Compañero, etc.), no solo a uno.*

**[2026-05-21] TL Games — sistema de versionamiento por juego** — Cada juego traducido tiene `versions/vX.X.XX/` con snapshot del ZIP + NOTAS.txt automáticas, `05_snapshot.py`, `06_delta.py` y `GAME_INFO.txt`. Patrón implementado primero en TheDemonLordsLover.
*Why: facilita actualizar traducciones al recibir nuevas versiones del juego, empaquetar por versión, y tener referencia de saves/binarios sin reconstruir el contexto desde cero.*
