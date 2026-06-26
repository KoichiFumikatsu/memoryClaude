---
name: Proyecto dashlife — finanzas personales (rebuild de KelsieApp, Fase 1)
description: PWA self-host de finanzas inteligentes. Stack, despliegue, modelo de datos, decisiones y trampas técnicas reusables.
type: project
---

# dashlife — Fumilinux

Rebuild de KelsieApp, **Fase 1 = solo finanzas**. PWA personal, single-user. Repo
`/home/kelsie/projects/dashlife`. Activo desde junio 2026; datos reales arrancan 2026-06-24.

## Stack y despliegue

- SvelteKit (Svelte 5 runes) + SQLite + Drizzle, adapter-node. Config del adapter en
  `vite.config.ts` (no hay `svelte.config.js`).
- Self-host en **Fumilinux tras Tailscale**: `https://fumilinux.tail7024a4.ts.net:8443`
  (`tailscale serve --https=8443 → 127.0.0.1:5173`). Login con `APP_PASSWORD=test1234`.
- Servicio: **systemd user** `dashlife.service` (`EnvironmentFile=.env`, HOST 127.0.0.1,
  PORT 5173, ORIGIN la URL de Tailscale, `ExecStart=/usr/bin/node build`).
  Reiniciar: `systemctl --user restart dashlife.service`. Logs: `journalctl --user -u dashlife.service`.
- Verificar: `npm run check` · `npm test` (node:test, puros) · `npm run build`.
- DB en `dashlife.db` (`DB_PATH`). Migraciones: `npx drizzle-kit generate`, se aplican al
  arrancar (`db/index.ts` migrate()).
- IA: **Claude Haiku** (`@anthropic-ai/sdk`, `output_config` json_schema) lee imágenes y
  correos no-bancarios; Ollama local quedó solo de respaldo (en CPU no servía). Avisos por
  **ntfy** (topic propio; hay que suscribirse en la app ntfy del celular).

## Modelo / decisiones

- Captura: **ingresos + suscripciones** automáticos por correo (IMAP App Password,
  `pollGmail` cada 15 min). Cobros de banco → estado `reminder` (no cuentan hasta
  registrarlos). Resto: foto de recibo o extracto.
- **Cuadre** (`/reconcile`): sube CSV **o foto de la lista** (Nu no exporta CSV → Claude
  visión la lee). Compras de Nu se importan como cuenta `credit` (deuda tarjeta); los
  "ingresos" del extracto de crédito se omiten (se ven desde el banco que paga). Idempotente.
- Dashboard = **KPIs explicativos** (cada cifra con frase): saldo, plata libre, te entró,
  gastaste, cobros fijos por guardar, deuda tarjeta, tope (te quedan / te pasaste).
  "Te pasaste" se mide contra el **tope mensual**. Plata libre = saldo − cobros fijos − pendientes.
- Cobro fijo "**pago de tarjeta**" (`tracksCardDebt`): monto = deuda actual, día fijo, y
  aviso **un día antes** por ntfy (cron diario `remindUpcoming`, anti-repetición `remindedDue`).
- **Foto/factura**: el FAB pregunta **cámara** (tomar foto) vs **galería** (elegir
  imagen/pantallazo de factura virtual) — dos `<input>`, solo el de cámara lleva
  `capture="environment"`. Luego se elige banco y Claude la lee.
- **Offline**: foto sin internet → cola IndexedDB (`src/lib/outbox.ts`); se procesa al
  reconectar. Lo usan `/capture` y el FAB foto del inicio.
- Cobros fijos tienen selector de dueño (Yo/otros) como el alta manual.
- Owners: `Yo` y otros (Mar). `own_names` (auto-traspaso): KOICHI FUMIKATSU,
  LUIS IGNACIO BURITICA. `credit_accounts`: Nu. Trabajo paga Anthropic/Claude/GitHub (excluidos).

## Trampas técnicas (reusables)

- `BODY_SIZE_LIMIT=15M` en `.env`: las fotos de celular pesan varios MB y adapter-node
  corta en 512 KB por defecto.
- Encoger imágenes en el navegador (canvas → JPEG) antes de subir: más rápido y evita el límite.
- Claude visión con montos **COP**: el punto "." y la coma "," son separadores de **MILES**,
  NO decimales. Hay que decírselo explícito o devuelve 38.9 en vez de 38900.
- `{#each}` con clave compuesta (fecha+comercio+monto) puede generar **duplicados** → Svelte
  rompe la lista al hidratar (los checkboxes desaparecen y el form va vacío). Usar el índice
  como key en listas estáticas que no se reordenan.
- Form actions que renderizan resultado rico: si el cliente no aplica `use:enhance`, usar
  envío nativo (sin enhance) para que el server haga SSR del resultado.

## Iconos (sin emojis)

- 2026-06-26: se **quitaron todos los emojis** de la UI (cumple `feedback_no_emojis`) y se
  reemplazaron por **Bootstrap Icons** (dependencia `bootstrap-icons`, fuente importada en
  `+layout.svelte`). Componente `src/lib/Icon.svelte` (`<Icon name="camera-fill" />`, sin el
  prefijo `bi-`). En mensajes/toast dinámicos se quitó el emoji y queda solo texto. La fuente
  woff2 entra al build (cacheada por el service worker → funciona offline).
