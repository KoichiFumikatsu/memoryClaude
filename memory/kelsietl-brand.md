# Kelsie TL — Brand Project

## Qué es

Marca pública bajo la que Koichi publica traducciones EN→ES de videojuegos. Proyecto separado del workspace técnico `tlgames` (el repo técnico NO se renombra).

**Repo local:** `/home/kelsie/projects/kelsietl` (git, rama main, remote pendiente)
**Figma:** https://www.figma.com/design/9M9cnBxZDXsHbig2SExukK

## Decisiones tomadas (2026-05-25)

- Nombre de marca: **Kelsie TL** (handle en todas las plataformas)
- TL Games (repo técnico) NO se renombra — son proyectos distintos
- Proyecto de marca = repo aparte en `/home/kelsie/projects/kelsietl/`

## Sistema visual

| Token | Hex |
|---|---|
| Background | `#0F0F23` |
| Card | `#1E1C35` |
| Primary | `#7C3AED` |
| Secondary | `#A78BFA` |
| Accent | `#F43F5E` |
| Foreground | `#E2E8F0` |
| Muted | `#94A3B8` |

- Heading: Fredoka Bold/Semi Bold (fallback Nunito Extra Bold)
- Body: Nunito Regular / Semi Bold
- Paleta validada con UI/UX Pro Max (Gaming profile)

## Assets Figma

| Asset | Node | Estado |
|---|---|---|
| Avatar 400×400 | `1:2` | Temporal — pendiente logo principal |
| Banner Twitter/X | `1:6` | Listo |
| Post Release 1080×1080 | `1:16` | Template listo |

## Sitio web — `/home/kelsie/projects/kelsietl/web/kelsietl-web`

Construido 2026-05-25. Stack: Next.js 16.2.6 + Tailwind v4 + TypeScript + Supabase (mock fallback activo mientras no haya proyecto Supabase disponible).

- Dev: `npm run dev` → localhost:3000
- Admin: `/admin`, contraseña en `.env.local`
- Supabase bloqueado: free tier tiene 2 proyectos activos (household-os + DnDev). Hay que pausar uno para conectar.
- **Tailwind v4 trampa clave:** `@import url()` fonts ANTES de `@import "tailwindcss"` en globals.css.
- **Fix visual:** cards usaban `bg-card` ≈ `bg-bg` → info section invisible → solución: shadow + gradientes por engine como placeholder thumbnail.
- **Bug corregido:** admin sidebar aparecía en el login form → layout.tsx ahora condicional por auth state.

## Estado del proyecto

- [x] Identidad visual completa
- [x] Assets base en Figma
- [x] Repo local + estructura de carpetas
- [x] Sitio web completo (público + admin, mock data)
- [ ] Supabase conectado (pausar household-os o DnDev)
- [ ] Logo principal → avatar derivado
- [ ] GitHub remote
- [ ] Cuentas: Twitter/X, Discord, Patreon
- [ ] Deploy en Vercel
