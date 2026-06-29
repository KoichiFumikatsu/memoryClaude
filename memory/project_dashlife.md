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

## Módulo Despensa / inventario (Fase 1 — CÓDIGO COMPLETO, FALTA DESPLEGAR)

Estado al 2026-06-29 (sesión interrumpida por batería). Plan aprobado en
`/home/kelsie/.claude/plans/ahora-ese-dash-life-precious-castle.md`.

**Qué hace (Fase 1 núcleo):** inventario de mercado. Cargar items desde **foto de factura**
(Claude visión extrae líneas → **pantalla de confirmación** edita/vincula/crea), marcar consumo
con botones **−/+**, promedio/día + días-restantes (informativo), y **aviso ntfy** cuando un item
llega a su **umbral fijo**. NO toca finanzas (gasto se registra aparte). Decisiones del usuario:
umbral fijo por item, botones rápidos, confirmación de factura, despensa separada de finanzas.

**Diseño forward-compat (NO construido aún):** lista de compras + reconciliación recibo↔lista/finanzas
(`pantry_purchases.receipt_id`), calorías (`pantry_items.calories_per_unit`), recetario.

**Archivos CREADOS:**
- `src/lib/server/pantry-math.ts` + `.test.ts` — `pantryStats(rows,currentQty,threshold,today,windowDays=14)`: avgPerDay (span inclusivo desde 1er consumo), daysLeft, isLow. 5 tests verdes.
- `src/lib/server/pantry.ts` — `pantryLowStockCheck()` con anti-spam `notified_low` (patrón remindedDue: avisa 1×/día, se re-arma al reponer).
- `src/routes/api/pantry-receipt/+server.ts` — sube foto, Claude lee líneas, devuelve `{items,ref}` SIN persistir.
- `src/routes/pantry/+page.{server.ts,svelte}` — lista, −/+ (action `consume`, qty firmado), `restock`, `setThreshold`, `addItem`, `removeItem` (soft-delete `active=false`).
- `src/routes/pantry/capture/+page.{server.ts,svelte}` — foto → confirmación (name, empaques, unid/empaque, precio/empaque) → action `confirm` suma `quantity*unitsPerPack` al stock o crea item; precio/u = precio empaque / unidades.
- `drizzle/0008_outgoing_the_phantom.sql` — migración de las 3 tablas (generada, **se aplica al reiniciar el servicio**).

**Archivos EDITADOS:** `src/lib/server/db/schema.ts` (3 tablas + tipos `PantryItem/PantryPurchase/PantryConsumption`),
`src/lib/server/claude.ts` (`PANTRY_IMG_SYSTEM/SCHEMA` + `parsePantryReceiptImageClaude`, max_tokens 3000),
`src/lib/server/ollama.ts` (wrapper `parsePantryReceiptImage` Claude/Ollama), `src/lib/server/cron.ts`
(llama `pantryLowStockCheck` en `dailyChecks`), `src/routes/+layout.svelte` (nav "Despensa").

**Tablas (snake_case):** `pantry_items`(name,unit,category,current_qty,low_threshold,owner,notified_low,calories_per_unit?,active,created_at),
`pantry_purchases`(item_id,qty,pack_qty,units_per_pack,unit_price?,currency,date,source,receipt_id?,created_at),
`pantry_consumption`(item_id,qty[firmado],date,created_at).

**Verificación hecha:** `npm run check` 0 errores · `npm test` 19/19 · `npm run build` OK.

**⚠️ PENDIENTE AL RETOMAR (en orden):**
1. **Desplegar**: `cd /home/kelsie/projects/dashlife && npm run build && systemctl --user restart dashlife.service` (el restart aplica la migración 0008 que crea las tablas).
2. **Smoke en la PWA** (cerrar/reabrir para refrescar el service worker): /pantry → alta "Huevos AAA" (unit huevo, umbral 8) y "Leche deslactosada" (bolsa, umbral 2); /pantry/capture con foto de factura → confirmar líneas (units_per_pack=30 huevos) → verificar stock +30/+6 y fila en `pantry_purchases`; botón − varias veces → ver "~N/día, ~M días"; bajar al umbral → llega ntfy "Despensa baja".
3. Marcar task #5 y #6 completas. Considerar `git`/commit del repo dashlife si el usuario lo pide (no se ha commiteado).

## Fixes de finanzas de esta sesión (ya desplegados)

- **Doble resta del saldo**: `accountOf` (en `recurring-math.ts`) decidía crédito por substring del
  COMERCIO ("Minimarket NUEVA" matcheaba "nu"); ahora decide por **banco exacto** (`bank === 'nu'`).
  Las compras de Nu son `credit` (deuda, no tocan saldo). Test en `classify.test.ts`.
- **Botón "Ya lo pagué"** (recurrentes): `markPaid` usaba `nextMonthlyDue(día, hoy)` → no avanzaba si
  marcabas antes del vencimiento (hoy 29 < vence 30). Fix: `from = max(hoy, next_due)`. Botón también
  en dashboard "Lo que se viene" (postea cross-route a `/recurring?/markPaid`). Test en recurring-math.test.ts.

## Iconos (sin emojis)

- 2026-06-26: se **quitaron todos los emojis** de la UI (cumple `feedback_no_emojis`) y se
  reemplazaron por **Bootstrap Icons** (dependencia `bootstrap-icons`, fuente importada en
  `+layout.svelte`). Componente `src/lib/Icon.svelte` (`<Icon name="camera-fill" />`, sin el
  prefijo `bi-`). En mensajes/toast dinámicos se quitó el emoji y queda solo texto. La fuente
  woff2 entra al build (cacheada por el service worker → funciona offline).
