# User Profile — Koichi Fumikatsu

## Identity
- **Name**: Koichi Fumikatsu
- **Apodo gaming**: Kelsie
- **Email**: fumikatsu.koichi@gmail.com
- **Celular**: 302 244 9235
- **Ubicación actual**: Tokio, Japón (desde ~2024, ya 2 años). Usar esto para zona horaria (JST/UTC+9), idioma local y referencias geográficas.
- **Origen**: nacido en Tokio, Japón. Vivió en Colombia (Cali) de los 5 a los 26 años; regresó a Japón hace ~2 años.
- **REGLA CV / hojas de vida**: en CVs y perfiles laborales mantener ubicación = **Colombia** (sigue empleado por empresa colombiana, Grupo AZC). NO poner Japón en material profesional.
- **LinkedIn**: existe, sin actualizar — pendiente
- **GitHub**: https://github.com/KoichiFumikatsu (repos desactualizados; Localfy no está subido — limpiar antes de enviar CV)

## Languages
- Español: nativo
- Inglés: A2 — bloqueador número 1 para mercado remoto internacional (requiere B2 mínimo)
- Japonés: A1

## Education
- Ingeniería Multimedia — Universidad Autónoma de Occidente, Cali · 7mo semestre, en curso
- Gerencia de Proyectos — Politécnico de Colombia · 2025 (cert. estimada Mayo 2026)
- Desarrollo de Software — MINTIC (sin peso internacional)
- Desarrollo de Software — Parquesoft (sin peso internacional)

## Employment History

**Grupo AZC S.A.S. / Bilingüe Legal Services — Cali y Medellín**
*Líder IT / Director de Desarrollo de Software* · Agosto 2022 – Presente
- Ingresó como único técnico IT, 1 sede Cali, 20 usuarios
- Escaló a equipo de hasta 4 ITs cubriendo 7 sedes y 200+ usuarios
- AZC: firma de abogados. Bilingüe: vende servicios legales de inmigración a firmas estadounidenses
- Cronología de expansión:
  1. Inicio: 1 sede Cali, 20 usuarios, solo operativo
  2. Año 1: montaje completo red Medellín (cableado, DVR, Omada, APs, switches)
  3. IT #2 Cali; Koichi a Medellín para 2da sede
  4. IT #3 contratado; nueva sede Cali y 3ra sede Medellín
  5. IT #4 contratado (duró 2 meses); IT #2 a Medellín
  6. Practicante SENA; 3ra sede Cali
  7. 4ta sede Cali; Soporte Nivel 3 contratado
  8. Recorte presupuestal (alza dólar): actualmente 3 ITs para 7 sedes
- El equipo actual solo sabe soporte operativo; no maneja Linux, dominios ni especialización de Koichi

**Soporte IT Remoto — Siesa**
*Soporte L1/L2 ERP* · Marzo 2021 – Diciembre 2021 · Remoto

**Operario Multifuncional — Pizzería El Callejón**
Cali · 2016 – 2021
*Gap enero–julio 2022: búsqueda activa en IT; poca experiencia formal dificultaba contratación*

## Skills & Expertise

**Redes:** TP-Link Omada SDN, multi-WAN (hasta 4 ISPs), VLANs, NAT, DNS, cableado estructurado
**Servidores:** Linux Ubuntu, Nginx, Nextcloud, Hestia Control Panel, RAID 1
**VoIP:** Grandstream UCM 6301/6308, SIP trunking, GDMS cloud
**Backend:** C# .NET 8, PHP 8 (MVC vanilla), MySQL, REST APIs, JavaScript vanilla
**Frontend/Cloud:** Next.js 16, React 19, TypeScript strict, Tailwind CSS 4, Alpine.js, Supabase, Vercel
**Mobile:** Flutter (Dart) — iOS + Android simultáneo (stack de Compañero)
**Email:** MX, DMARC, SPF, gestión de dominios
**Videovigilancia:** DVR, instalación y configuración de cámaras
**Herramientas:** GitHub, GitHub Actions, VS Code + Claude Code, Omada REST API
**Automatización local (desde 2026-05-19):** Ollama, n8n, Docker, tlgames-qa, tlgames-pipeline, tlgames-versions

## Hardware — Fumilinux (laptop personal)

- **Modelo:** ASUS Vivobook X1404VA
- **CPU:** Intel Core i7-1355U (13th Gen, 10 núcleos / 12 hilos, hasta 5.0 GHz)
- **RAM:** 24 GB DDR4 3200 MHz — 8 GB soldados + 16 GB SO-DIMM · **no ampliable** (un slot libre ya ocupado)
- **SSD:** SK Hynix PC711 NVMe 512 GB
- **GPU:** Intel Iris Xe Graphics (96 EU, 1300 MHz) — comparable a RX 560 · sin soporte CUDA/HIP útil
- **OS:** Ubuntu 24.04.4 LTS · Kernel 6.17 · GNOME en Xorg

**Optimizaciones aplicadas (2026-05-21):**
- TLP instalado → governor auto: `balance_performance` en AC, `balance_power` en batería
- Swappiness: 60 → 10 (persistente en `/etc/sysctl.d/99-performance.conf`)
- SSD: scheduler `none` + TRIM activo (ya estaba correcto)
- EasyEffects **Flatpak** (Flathub) instalado con RNNoise para cancelación de ruido en mic — preset `microfono-limpio` en `~/.var/app/com.github.wwmm.easyeffects/data/easyeffects/input/` — autostart en `~/.config/autostart/easyeffects.desktop` — dispositivo virtual: `easyeffects_source` (seleccionar en Discord como entrada)
  - **Nota:** el paquete `apt` de Ubuntu 24.04 viene compilado sin RNNoise. Siempre usar la versión Flatpak.

## Stack de automatización local (Fumilinux)

- **Ollama** — `systemctl start/stop ollama` · endpoint `http://localhost:11434` · modelo `llama3.2:3b` (2.0 GB, CPU only — sin GPU)
- **n8n 2.20.11** — Docker (`sudo docker start/stop n8n`) · UI `http://localhost:5678` · login `fumikatsu.koichi@gmail.com` / `TlGames2026!`
- **MCP `ollama-mcp`** — scope usuario en `~/.claude.json` · activo en todas las sesiones Claude Code · herramientas: `ollama_generate`, `ollama_chat`, etc.
- **tlgames-qa** — systemd · `http://localhost:8765/qa` · QA semántico Ren'Py via Ollama
- **tlgames-pipeline** — systemd · `http://localhost:8766` · pipeline completo (detect engine → translate → lint → QA); Ren'Py automatizado, Unity/RPGMaker requieren manual
- **tlgames-versions** — systemd · `http://localhost:8767` · monitor versiones itch.io/F95Zone; DB en `tools/.versions.json`
- **Carpeta de juegos:** `/home/kelsie/Downloads/Games h/` (sistema en inglés — no "Descargas")

## Hardware — Torres Windows

**Torre 1 (principal para AI/GPU):**
- GPU: AMD RX 570 4GB (Polaris, GFX8 / gfx803) — confirmado por vulkaninfo (4294967296 bytes VRAM)
- OS: Ubuntu 24.04 LTS — instalado y clonado desde Fumilinux el 2026-05-26
- IP LAN: 192.168.12.7, SSH puerto 22, usuario `kelsielinux` (trampa: NO es `kelsie`)
- ROCm 6.4 instalado, RX 570 detectada, `HSA_OVERRIDE_GFX_VERSION=9.0.0` activo
- Stack completo: Claude Code, Docker CE + n8n, Ollama + llama3.2:3b, Wine, Steam, Blender, etc.
- Tailscale activo — Torre 1: 100.67.216.43, Fumilinux: 100.116.50.47
- Escritorio remoto: GNOME Remote Desktop RDP puerto **3389** (verificado 2026-06-04; la nota previa de "3390" era stale), Remmina desde Fumilinux (perfil "Torre Linux": `100.67.216.43:3389`)
- Pendiente: rclone OAuth, n8n workflows, Vector venv

**Torre 2:**
- GPU: AMD RX 6500 XT 4GB (RDNA2, Navi 24) — **candidata a venta/upgrade**
- CPU: AMD Ryzen 5 5500
- Board: Gigabyte B450M DS3H V2 (PCIe 3.0 x16)
- OS: Windows
- Limitación: PCIe x4 GPU + bus 64-bit → stutter notable en gachas (Endfield, Honkai, ZZZ)
- Upgrade objetivo: RX 6600 8GB (mínimo) o RX 6700 XT 12GB (ideal, también sirve para AI)

**Interés pendiente (2026-05-22):** Instalar Linux en Torre 1 (RX 570) para AI generation local + gaming con Proton. Ya investigó compatibilidad Proton/Steam con juegos Windows en Linux — sin bloqueos conocidos.

**Gaming Windows:** Juega gachas — Arknights Endfield, Honkai Star Rail, ZZZ y similares. Usa Torre 2 para gaming Windows.

## Gaming Setup (Linux)
- **Ankama Launcher (Dofus 3):** AppImage en `/home/kelsie/Downloads/Dofus 3.0-Setup-x86_64.AppImage`
- Requiere flag `--no-sandbox` (Electron sandbox issue en Linux)
- .desktop en `~/.local/share/applications/dofus.desktop` y `~/Desktop/dofus.desktop`
- **Steam + Proton:** instalado, Proton activo
- **GE-Proton10-34:** instalado en `~/.steam/root/compatibilitytools.d/GE-Proton10-34` (2026-05-21)
- **Lutris 0.5.22:** instalado vía PPA lutris-team (2026-05-21) — para juegos itch.io y no-Steam
- **Interés en Hoyoverse (Linux):** Genshin/HSR/ZZZ requieren launchers especiales (`an-anime-team` en GitHub) — anti-cheat HoYo-PROTECT no compatible con Proton/Lutris estándar

## Projects in Production

| Proyecto | URL | Stack | Rol |
|---|---|---|---|
| AZCKeeper | keep.azclegal.com | C# .NET 8, PHP 8, MySQL | Desarrollador principal |
| DSL | dsl.azc.com.co/diagnostic.php | PHP | Desarrollador principal |
| Localfy | localfy.com.co | PHP, JS vanilla | Backend + estructura visual (freelance) |
| KelsieApp | kelsieapp.vercel.app | Next.js 16, Supabase, Vercel | Arquitectura y dirección; implementación con IA |
| Altergeist | danielos8900.itch.io/altergeist | — | Codesarrollador (Platzi Game Jam 2020) |
| Voice Assistant | /home/kelsie/voice-assistant/ | Python, Claude API, faster-whisper, gTTS, Google Calendar | Arquitectura y desarrollo con IA |
| Compañero | Pre-producción (sin URL) | Flutter, Supabase, Firebase, Rive | Implementador + sugeridor técnico (NO product owner); arte 3D clay render: Mar |
| TL Games | /home/kelsie/projects/tlgames (github.com/KoichiFumikatsu/tlgames) | Python + herramientas GUI (UABEA, BepInEx, Ren'Py SDK) | Proyecto personal: traducción EN→ES de videojuegos (Ren'Py, Unity, RPG Maker, GameMaker, Electron) |

**Pendiente:** Localfy no está en GitHub (código en local). Repos de GitHub desactualizados.

**Nota KelsieApp:** Koichi diseñó la arquitectura, estructura de módulos y visión de producto. La implementación fue dirigida por él con asistencia de IA. En entrevistas: "arquitectura diseñada y dirección de implementación con asistencia de IA" — no presentar como desarrollo propio completo.

## Career Plan

**Ruta elegida:** liderazgo técnico (no pivot a IC remoto).

**Roles objetivo — Liderazgo (prioridad):**
- IT Manager / Gerente de TI
- Coordinador de IT / Líder Técnico IT
- Director de TI (SMBs LATAM)

**Roles objetivo — IC (alternativo):**
- IT Infrastructure Engineer / Systems Administrator Jr/Mid
- Network Support Engineer / IT Support Specialist L2

**Plan 18 meses (desde Mayo 2026):**

| Plazo | Acción |
|---|---|
| 0–6 meses (Mayo–Nov 2026) | Resolver dependencia operativa del equipo. Documentar procesos. Empezar métricas. Actualizar GitHub y LinkedIn. |
| 6–12 meses (Nov 2026–May 2027) | Resultados medibles del equipo. Leer *The Manager's Path* y *An Elegant Puzzle*. Evaluar CompTIA. |
| 12–18 meses (May–Nov 2027) | Inglés a B2. Certificación CompTIA lista o en curso. Aplicar a roles remotos. |

**Prioridad única e irremplazable:** inglés B2. Todo lo demás es secundario.

**Plataformas:** Torre.ai, GetOnBoard, Workana, Talently, LinkedIn (LATAM) · Toptal, We Work Remotely, Remote.com (requieren B2+)

## Known Constraints
- Inglés A2 — mercado internacional requiere B2 mínimo
- Sin título universitario (7mo semestre) — filtro en empresas
- MINTIC/Parquesoft no tienen peso internacional
- Tendencia a abandonar formación formal (2 cursos de PM abandonados) — flagear si relevante
- Patrón doer-manager: absorbe trabajo técnico porque es más rápido que enseñar → cuello de botella permanente. Es consciente, trabajando en delegación estructural.
