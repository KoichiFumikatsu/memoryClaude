---
name: project_uhppoted
description: uhppoted access control system — migrado a servidor central azc.com.co (192.168.12.25) el 2026-05-29. Subdominio doors.azc.com.co
type: project
---
# Estado actual (2026-05-29)

**MIGRADO al servidor central:** `azc.com.co` (LAN `192.168.12.25`, NAT `186.145.239.174`). Fumilinux (192.168.12.168) ya **NO** corre uhppoted — services stopped + disabled.

## Tequendama: transporte migrado a Tailscale (2026-06-12) — REEMPLAZA uhppoted-tunnel

**Motivo:** el relay `uhppoted-tunnel.exe` corría en consola en el mini PC Windows de Tequendama y **se cayó** → las 4 placas Teq quedaron incomunicadas (sin conexión en 60443, solo escaneos de bots). Se reemplazó el túnel por **Tailscale subnet routing**.

- **Tailnet:** cuenta `techsupport@`. Nodos: `azc-doors-server` = `100.105.130.73` (server, tailscale 1.98.4), `DESKTOP-MF3LKIV` = `100.115.211.103` (mini PC Tequendama relay, LAN `192.168.12.209`).
- **Server install:** apt global roto (keys nginx/anydesk + `co.archive.ubuntu.com` inalcanzable) → instalar dirigido solo del repo tailscale (`apt-get update -o Dir::Etc::sourcelist=sources.list.d/tailscale.list ...` luego `apt-get install -y tailscale`). `tailscale up --accept-routes`. **`tailscale set --accept-dns=false`** OBLIGATORIO (el server corre `named`/BIND9; Tailscale le pisaba el DNS).
- **Subnet router (mini PC):** `tailscale up --advertise-routes=192.168.14.12/32,192.168.14.13/32,192.168.14.125/32,192.168.14.150/32` — solo los **4 /32** de las placas (quirúrgico: evita el solape `192.168.12.0/22` entre Palmetto y Tequendama, que son la MISMA subred). Rutas **aprobadas en consola**. En el server caen en `ip route show table 52` + `ip rule` 5270.
- **`bind.address = 0.0.0.0` en `uhppoted.conf` — CRÍTICO:** con `192.168.12.25` (eno1) el source NO casa con la salida por `tailscale0` → **las 4 dan timeout**. Con `0.0.0.0` el kernel elige source por destino (IP tailscale para Teq, eno1 para Palmetto). Palmetto no se afecta.
- **`.address` de las 4 placas en conf = IP real** (`192.168.14.13/.125/.150/.12`), ya NO `192.168.12.25:60010`. Backup `uhppoted.conf.bak.tailscale-20260612-145518`.
- **Warmup persiste:** el path va por **DERP (relay Miami ~125ms)**, no directo → el cold-start tras idle **dropea los primeros paquetes**. Un `get-device` solo tras idle falla; **ráfaga** (varios paquetes rápidos) entra. `teq-keepalive.service` (15s) mantiene las 4 calientes. **Pendiente:** conexión **directa** Tailscale (abrir UDP 41641 en firewall del mini PC) eliminaría el warmup.
- **Verificado 2026-06-12:** las 5 placas responden con conf de producción; poller drena backlog sobre Tailscale (cursores avanzan). `uhppoted-tunnel-tequendama.service` **stop + disable**. **Pendiente Koichi:** cerrar port-forward 60443 en Omada Palmetto.
- **Re-integración al panel nativo: PROBADA y REVERTIDA (2026-06-12) — NO viable.** Con IPs reales distintas el demux SÍ se resuelve (`invalid controller ID: 0`), pero **el loop de poll de httpd se cuelga igual**: su polling concurrente/ráfaga (varios requests por placa por ciclo) ahoga el path serializado a las Teq (los UT0311 atienden 1 request a la vez sobre Tailscale) → 60-70 timeouts/min y **Palmetto queda STARVED (no refresca en 5 min)**. Más keepalive lo EMPEORA (compite por las placas); httpd-solo también falla. **CLI secuencial a las 5 da OK intento 1 siempre** → la red está perfecta, el cuello es el diseño del poll de httpd. **Conclusión: Opción B sigue siendo la topología correcta** (Teq fuera de `controllers.json`, poller secuencial dedicado), ahora corriendo **sobre Tailscale**. El demux ya no es el motivo de Opción B; el motivo es que httpd no tolera placas latentes/serializadas en su loop. Para panel nativo real haría falta cambio upstream en httpd (aislar poll por controlador / timeout-concurrencia configurable). Backups del intento: `controllers.json.bak.ts-reintegrate-*`, `events.json.bak.ts-reintegrate-*`, `*.bak.cursorjump2-20260612-*`.
- **`ACL 222451671 ... deleted:112` en logs httpd = BENIGNO y pre-existente** (desde 2026-06-11). Es el diff dry-run de ACL: `cards.json`=352 (240 Palmetto + 112 solo-Teq); `acl.grl` determina que las 112 Teq NO van en Palmetto → aparecen como "deleted" en la comparación. **NO se aplica** (sin auto-sync; 2853 ocurrencias/24h y Palmetto conserva sus tarjetas). No tocar.

## Reconciliación de tarjetas — cosecha 2026-06-12 sobre Tailscale

Cosecha por unión confiable en `/root/harvest/ts-20260612-164606/` (`union_harvest_ts.sh`: 5 lecturas get-cards/placa, unir nros; las lecturas sueltas vienen PARCIALES, no confiar en un solo get-cards).

- **Conteos reales:** Palmetto(222451671)=**240** (estable). Tequendama es **sistema replicado de 352**: .125(225088590) y .12(425036574) **idénticas a 352**; .150(423150802)=341 (subconjunto, le faltan 11); .13(223205300)=solo 190 legibles (flaky v6.62) pero todo lo leído ∈ 352.
- **Palmetto(240) ⊂ Teq(352):** solo-Palmetto=0, en-ambas=240, solo-Teq=112. Unión maestra=**352**. (Cambió vs 02-jun que decía 115 compartidas → entre el 02 y el 12-jun alguien empujó las tarjetas de Palmetto a Teq.)
- **🔑 SEGURIDAD: ninguna placa tiene cards fuera de la unión 352** → publicar el set de 352 **NO borra ninguna tarjeta de ninguna placa** (`deleted:0` garantizado a nivel de membresía). El `.13` flaky NO bloquea: es réplica (todo ∈ 352, y .125/.12 dan las 352 completas) y `load-acl` solo agrega, nunca borra si el set ⊇ lo que tiene.
- **Panel `cards.json` YA tiene las 352, todas con grupo** (mayoría en Empleados 0.5.11). 7 grupos: Admin 0.5.1, Retirados 0.5.2, Invitados 0.5.3, Directivos 0.5.8, CasoEspecial 0.5.9, ServGen 0.5.10, Empleados 0.5.11. → roster completo para gestionar permisos en el panel SIN tocar hardware.
- **Regla uhppoted:** una tarjeta existe en una placa **solo si tiene ≥1 puerta concedida** ahí — NO hay estado "presente sin acceso" en hardware. Por eso los permisos van ANTES del push. Decisión de Koichi (2026-06-12): cargar todas y tratar permisos después → el roster del panel (352) ya lo permite; el push al hardware espera a que defina permisos + `compare-acl deleted:0` + revisar lectores entrada/salida compartidos (revocar ahí ENCIERRA).

## httpd panel AUTO-SINCRONIZA a Palmetto (NO a Teq) — descubierto 2026-06-12

**El panel httpd aplica los cambios de grupos/cards AUTOMÁTICAMENTE a los controladores que tiene en `controllers.json`** (cada refresh ~30s: logs `ACL <serial> ... / ACL compare`). Koichi editó el grupo **Empleados → todas las puertas**; como las 352 cards están en Empleados, **Palmetto quedó 352 × `Y Y Y Y`** (todos abren las 4 puertas) sin que nadie tocara publish. Time profiles sobrevivieron (Profile 2 intacto). Confirmó: "no hay problema si todos tienen todo".

**CLAVE — split de gestión por Opción B:**
- **Palmetto** está en `controllers.json` → el panel lo auto-sincroniza. Editar grupos en el panel = se aplica solo a Palmetto.
- **Las 4 Teq NO están en `controllers.json`** (Opción B) → el panel **no las toca**. Quedaron con su acceso **variado real** (ej. .12: 161 cards `Y Y Y Y`, 26 restringidas; .125 `Y Y N N`). Por eso tras el cambio Palmetto=todos-todo y Teq=variado → **inconsistentes**.
- **Gestionar/propagar acceso a Teq = vía publish/`load-acl` sobre Tailscale** (NO el panel). PERO el `api_publish`/`generate_acl_tsv` actual arma columnas de puerta desde `controllers.json` (solo Palmetto) → **el publish hoy NO incluye Teq**. Para centralizar de verdad hay que wirear el generador para incluir los 4 controladores Teq (desde conf + doors.json) aunque no estén en controllers.json. PENDIENTE.
- **Lecturas get-cards sueltas sobre Tailscale = PARCIALES** (.125 leyó 20/352, .12 187/352). Para cualquier op real usar cosecha por unión, no lecturas sueltas.

## Pestaña "Controladores" (visibilidad on-demand) — agregada 2026-06-12

Resuelve "ver los controladores sin tiempo real": en vez de meterlos en controllers.json (starva Palmetto), pull **secuencial on-demand**. En `schedule-manager`:
- `GET /api/controllers-status` (lee cache `/var/uhppoted/controllers-status.json`), `POST /api/controllers-refresh` (worker en background, daemon thread, recorre `CONTROLLERS_META` = las 5 placas 1×1 con reintentos warmup, escribe el cache incremental placa por placa). Independiente de httpd → no afecta Palmetto. Backup `schedule-manager.bak.controllers-*`.
- UI: pestaña "Controladores" en `/schedules/index.html` (tabla nombre/serial/estado/IP/firmware/tarjetas/actualizado + botón "Actualizar (1×1)" que pollea status cada 2s hasta terminar). Backup `index.html.bak.controllers-*`. Verificado HTTP 200 vía nginx (192.168.12.25:443).

## Publish ACL board-by-board (self-service para IT) — 2026-06-12

Se entrega a IT (no tendrán la cuenta de Koichi ni a Claude) → el publish debe ser un botón robusto, sin depender de nadie.

- **`api_publish` reescrito a async secuencial placa-por-placa.** `POST /api/publish` (opcional `{controllers:[...]}`, default `PUBLISH_ORDER`=las 5, `.150` última) lanza un worker en background; `GET /api/publish-status` da progreso por placa. UI: pestaña "Publicar" → botón pollea cada 2s y muestra tabla por placa (en cola/publicando/OK/FALLÓ + `+add ~upd -del =igual` + reintentos). Backups `schedule-manager.bak.pubseq-*`, `index.html.bak.pubseq-*`.
- **Por qué placa-por-placa:** `compare-acl`/`load-acl` golpean TODAS las placas concurrente → timeout en bloque sobre Tailscale (igual que httpd). Worker usa **conf aislado por controlador** (`_isolated_conf`: filtra `uhppoted.conf` a solo las líneas `UT0311-L0x.<serial>.*` — ojo: hay que filtrar TODAS, no solo `.address`, porque compare/load enumeran controladores por las **door-labels** del conf) + **TSV cortado** a sus columnas (`generate_acl_tsv(only_serial=...)`).
- **PINs (CRÍTICO):** Teq usa PIN; `load-acl`/`compare-acl` necesitan `--with-pin` (flag DESPUÉS del subcomando) o borran PINs. Mapa en `/etc/uhppoted/card-pins.json` (112 cards, cosechado por unión, mayoría `345678`). TSV con PIN = `Card,PIN,From,To,doors`. `generate_acl_tsv` lo emite. Las 240 de Palmetto son card-only (sin PIN).
- **Generador incluye Teq sin meterlas en controllers.json:** `/etc/uhppoted/acl-extra-controllers.json` (4 Teq con door-mappings 0.3.5-0.3.16) merge en `_controllers()`. Así el TSV tiene las 16 columnas pero httpd no las pollea (no starva Palmetto).
- **Bulk get-acl/load-acl sobre Tailscale = todo-o-nada:** si UN paquete de los 352 reads se cae, falla toda la operación. Por eso reintentos (`PUBLISH_RETRIES`: default 3, `.150`=8, `.13`/`.12`=12). **El `teq-keepalive` CONTIENDE con el bulk** (pinguea la misma placa que se está leyendo) → el worker **pausa `teq-keepalive` durante el publish y lo reanuda al final** (`systemctl stop/start`). Con el keepalive parado, `.12`/`.13` entraron en intento 1. Detección de éxito: NO confundir `errors:0` del summary con error real (chequear `i/o timeout` / `error: [`).
- **ESTADO FINAL 2026-06-12:** las **5 placas = 352 tarjetas, todas `Y` en todas sus puertas** (Teq igual que Palmetto, decisión de Koichi "todos tienen todo"). Empleados (0.5.11) concede las 16 puertas; las 352 están en Empleados. `compare-acl` previo: extraneous(delete)=0 en todas → nadie perdió acceso.
- Tooling manual: `/root/acl_seq2.sh compare|load <tsv>`, `/var/uhppoted/acl/candidate.tsv`, `/tmp/conf-per-ctrl/`, `/var/uhppoted/per-ctrl-conf/`.

## Servicios systemd activos en el server

- `uhppoted-httpd.service` — UI web, puertos **8543 HTTPS** y **8580 HTTP** (cambiados de 8443/8080 por conflicto con Apache de Hestia)
- `uhppoted-rest.service` — REST API, puerto **8444 HTTPS**
- `door-opener.service` — microservicio Python custom, puerto **8445 HTTPS**

## Acceso

- **URL pública (cuando DNS propague + LE listo):** `https://doors.azc.com.co/`
- **URL LAN (siempre disponible):** `https://192.168.12.25/` con Host header `doors.azc.com.co`
- **URL panel directo (saltea reverse proxy):** `https://192.168.12.25:8543/`
- Login admin: `admin / uhppoted` (PENDIENTE: cambiar)

## Reverse proxy en Hestia

- Vhost `/home/azcweb/conf/web/doors.azc.com.co/` (user `azcweb` por enforce_subdomain_ownership)
- nginx 443 → `proxy_pass https://127.0.0.1:8543`
- HTTP/80 → 301 a HTTPS
- robots.txt en `/home/azcweb/web/doors.azc.com.co/public_html/robots.txt` (Disallow /)
- Headers anti-bots: `X-Robots-Tag: noindex, nofollow, noarchive, nosnippet`, X-Frame-Options, Referrer-Policy

## Controlador (Sede 1)

- Device ID: **222451671**
- IP LAN directa: `192.168.15.10` (el server azc está en la misma /22, no necesita NAT/port forward)
- Listener actual: **`192.168.12.25:60001`** (set-listener actualizado el 2026-05-29)
- 240 tarjetas sincronizadas, 7 grupos válidos
- uhppoted-httpd **funciona por polling** cada 30s al controlador (no UDP push). UDP listener es para uhppoted-rest

## TLS

- CA self-signed regenerada el 2026-05-29 en `/etc/uhppoted/httpd/ca.cert` (válida 10 años) — usada por uhppoted-httpd (8543), uhppoted-rest (8444) y door-opener (8445) en el backend interno
- **Cert LE válido en nginx vhost** `/home/azcweb/conf/web/doors.azc.com.co/ssl/` — emitido con `certbot certonly --webroot` directo (Hestia `v-add-letsencrypt-domain` fallaba con 403 inexplicado). Expira 2026-08-27
- Auto-renewal: `/etc/letsencrypt/renewal-hooks/deploy/doors-azc-sync-hestia.sh` copia cert renovado a paths Hestia + reload nginx. `certbot.timer` activo.

## Reverse proxy completo en nginx vhost SSL (doors.azc.com.co)

`/home/azcweb/conf/web/doors.azc.com.co/nginx.ssl.conf` proxy a:
- `location /` → `https://127.0.0.1:8543` (uhppoted-httpd panel)
- `location /door-opener/` → `https://127.0.0.1:8445/` (microservicio Python, slash final = strip prefix)
- `location /schedules/api/` → `https://127.0.0.1:8446/api/` (microservicio schedule-manager)
- `location /schedules/` → `alias /home/azcweb/web/doors.azc.com.co/public_html/schedules/` (UI estática)
- `location = /robots.txt` → archivo físico

## Schedule manager (time profiles UI) — agregado 2026-05-30

Microservicio Python `/usr/local/bin/schedule-manager` escuchando `127.0.0.1:8446` (HTTPS con cert uhppoted), systemd unit `schedule-manager.service`. Wrapping `uhppote-cli` para CRUD time profiles + permisos de cards.

**Endpoints REST (vía `https://doors.azc.com.co/schedules/api/*`):**
- `GET /profiles` · `GET /profile/<id>` · `PUT /profile/<id>` · `DELETE /profile/<id>`
- `GET /card/<n>` · `PUT /card/<n>` · `GET /cards-list` · `GET /doors`
- `POST /bulk-assign` (cards, door, profile)

**UI:** `https://doors.azc.com.co/schedules/index.html` — estructura idéntica al panel (header, nav, CSS uhppoted.css), 3 sub-tabs: Profiles / Card individual / Asignación masiva. Tab "HORARIOS" inyectado por custom.js en el nav del panel.

**Parsing:** `get-time-profiles` retorna tabla (id, from, to, Mon-Sun Y/N, 3 pares HH:mm, linked). Mi parser lo convierte a JSON `{id, from, to, weekdays[], segments[[start,end]...], linked}`.

**Persistencia:** profiles viven SOLO en el controlador, NO en cards.json. Si alguien hace ACL Sync desde el panel uhppoted, **borra los profiles asignados a cards** porque cards.json no tiene field `profile`. Workaround: NO usar Sync ACL después de configurar profiles, gestionar TODO via la UI/CLI.

**Asignación card+profile:** `put-card 222451671 <card> <from> <to> "1:2,2,3,4"` — sintaxis door:profile, NO documentada pero funcional. Doors sin profile id van como Y (24/7).

### CRUD verificado end-to-end + fix delete (2026-06-02)

Probado contra el controlador real: **crear / leer / actualizar / asignar funcionan correctamente** desde la API (y por ende la UI). Update refleja bien multi-segmento (hasta 3) y cambios de días/horas.

- **DELETE estaba roto:** el UT0311 no tiene delete por profile (solo `clear-time-profiles` borra TODOS). El workaround sobrescribía con rango `2000-01-01:2000-01-01` → el controlador **rechaza el año 2000** (lo trata como sentinela nula): `ERROR: could not create time profile`. **Fix:** `api_delete_profile` ahora usa `2020-01-01:2020-01-02` (rango expirado válido) + `_is_deleted()` filtra ese sentinela de `GET /profiles` para que el profile "borrado" desaparezca de la UI. El id sigue ocupado en el hardware (tope 254, irrelevante); re-crear el id lo sobrescribe. Backup `schedule-manager.bak.delfix-*`.
- **`set-time-profile` rechaza año 2000** como fecha — usar >=2020 siempre.
- **Asignación masiva verificada:** `POST /bulk-assign {cards:[],door:1,profile:7}` lee perms actuales de cada card, sobrescribe solo esa puerta, hace `put-card`. Confirmado card puerta1 Y→7 y restaurada.
- **LIMITACIÓN multi-sede:** `CONTROLLER = '222451671'` está **hardcodeado** en `/usr/local/bin/schedule-manager` (línea 28). La UI de HORARIOS solo gestiona Sede 1. Para la 2da sede (controlador nuevo en montaje 2026-06-02) hay que parametrizar el controlador (selector de sede en UI + API).

### Arquitectura ACL elegida: "B-desde-panel" + generador/publish (2026-06-02)

**Decisión:** gestión central multi-controlador vía `uhppote-cli load-acl` (profile-aware), con `cards.json`/grupos del panel como fuente. NO se usa el "Synchronize ACL" del panel (pisa los time profiles). El panel sigue para alta de datos + monitoreo en vivo; el push a controladores lo hace load-acl con profiles intactos.

**Confirmado:** el TSV de ACL transporta el **profile-id en la celda de puerta** (no solo Y/N); `load-acl` lo round-trips (`get-acl`→`load-acl` da `unchanged`). Los UT0311 son autónomos: con el ACL cargado abren sin server/internet, usando su reloj interno (→ hay que sincronizar hora periódicamente o los horarios fallan). Eventos no se pierden sin red: buffer interno se recupera al reconectar.

**Door labels en `uhppoted.conf`** (necesarios para que get/load-acl tengan columnas de puerta): `UT0311-L0x.222451671.door.{1..4} = S1 Porteria | S1 Puerta 2 | S1 Puerta 3 | S1 Puerta 4`. Prefijo S1 = Sede 1; labels deben ser únicos cross-controlador. Backup `uhppoted.conf.bak.doorlabels-*`.

**Generador + publish en schedule-manager** (nuevos endpoints):
- `GET /api/role-profiles` · `PUT /api/role-profiles` — mapa rol→permiso en `/etc/uhppoted/role-profiles.json`. Valor por rol: `Y` (24/7), `N` (sin acceso) o id de profile. Default todo `Y` (Retirados `N`).
- `GET /api/generate-tsv` — preview del TSV generado desde `cards.json` + `groups.json` + `controllers.json` + door labels del conf. Por puerta: `N` si ningún grupo de la card la concede; si la concede, el profile del rol (prioridad `ROLE_PRIORITY = Directivos>Administrativos>ServGen>Empleados>Invitados>CasoEspecial>Retirados`).
- `POST /api/publish` — genera TSV, lo guarda versionado en `/var/uhppoted/acl/published-<ts>.tsv` + `latest.tsv`, corre `load-acl`. **Publicar con default todo-Y dio `updated:0 added:0 deleted:0` — el generador reproduce el ACL actual fielmente, no cambia nada hasta asignar un profile a un rol.** Backup `schedule-manager.bak.aclgen-*`.

**UI:** nueva pestaña "Publicar" en `/schedules/index.html` (editor del mapa rol→profile, "Previsualizar TSV", "Publicar a controladores" con confirm). Backup `index.html.bak.publish-*`.

**Multi-controlador (estado):** `publish`/`load-acl` YA es multi (itera `controllers.json` + conf), así que al registrar el controlador de la 2da sede el push lo incluye. Falta para multi-sede completo: (1) door labels del nuevo controlador en conf, (2) crear profiles con **mismos ids** en cada controlador — el CRUD de profiles del schedule-manager sigue con `CONTROLLER='222451671'` hardcodeado (línea 28), (3) cron `set-time` por controlador para el reloj.

### Distribución real de cards por rol (2026-06-02)

7 grupos con acceso por puerta definido, pero **240/240 cards están en "Empleados"** (1 también en Administrativos). Los grupos Directivos/Servicios Generales/Invitados existen con door-access configurado pero **sin cards asignadas**. OIDs: Administrativos=0.5.1, Retirados=0.5.2, Invitados=0.5.3, Directivos=0.5.8, Caso Especial=0.5.9, Servicios Generales=0.5.10, Empleados=0.5.11. Door-access por grupo: Administrativos y Servicios Generales = P1-P4; Directivos/Empleados/Invitados = solo P1 (Portería); Caso Especial y Retirados = []. Puertas: P1=(P) Portería, P2, P3, P4 (OIDs 0.3.1-0.3.4).

## SSH público al server (habilitado 2026-05-30)

Port forward Omada `22 ext → 192.168.12.25:22` + iptables rule via `v-add-firewall-rule ACCEPT 0.0.0.0/0 22 TCP SSH_WAN`. Acceso desde cualquier red con clave SSH. Fail2ban-SSH activo (banea brute-force).

```bash
ssh root@186.145.239.174       # acceso público
ssh root@192.168.12.25         # acceso LAN local (sigue funcionando)
```

## custom.js — patches agregados 2026-05-30

`/opt/uhppoted-httpd-custom/httpd/html/javascript/custom.js` extendido con:
1. `_customSortDescByTimestamp(tbody)` — ordena events/logs descendente por `input.timestamp` value. Skip si ya está ordenado (evita mutaciones inútiles).
2. `_customWatchAndSort(selector)` — MutationObserver con `suppressing` flag para evitar auto-trigger loop. WeakSet `_customWatchedTbodies` evita duplicar observers.
3. `_customInjectScheduleTab()` — agrega `<li class="custom-schedules"><a href="/schedules/">HORARIOS</a></li>` al `<nav><ul>`. Idempotente.
4. `_customWatchNav()` — re-inyecta tab si el nav se reconstruye.

**custom.js solo se carga en páginas que tienen `{{template "custom.js" .}}` dentro de `<script type="module">`.** El template SOLO contiene imports + window assignments — NO incluye las tags `<script>`, tiene que ir DENTRO del module existente. Páginas que originalmente NO lo cargaban: overview, controllers, logs, users — se agregó manualmente.

## Bug controllers.html "texto plano" — RESUELTO 2026-06-02

**Síntoma:** páginas del panel (controllers, cards, events, logs, users...) se renderizaban como HTML texto plano en el browser, no como página. Reproducible en incógnito (NO era cache).

**Causa raíz:** el gzip interno de **uhppoted-httpd** (binario Go, puerto 8543) comprime respuestas sobre un umbral de tamaño (~12-16KB) y **al comprimir borra el header `Content-Type`**. Go auto-detecta el content-type del primer `Write`, pero con gzip el primer Write son bytes mágicos `1f 8b`, así que el type sale vacío. nginx pasa esa respuesta pre-gzipeada sin content-type. Con `add_header X-Content-Type-Options nosniff` activo en el vhost, el browser se niega a sniffear y muestra texto plano. Páginas chicas (doors.html, 12KB) quedaban bajo el umbral → no gzip upstream → conservaban content-type → renderizaban bien. Por eso doors SÍ y controllers NO.

**Diagnóstico decisivo:** `curl -b cookie -H 'accept-encoding: gzip' https://127.0.0.1:8543/sys/controllers.html` devuelve `content-encoding: gzip` SIN `content-type`; doors.html devuelve `content-type: text/html` SIN gzip.

**Fix aplicado (solo nginx, sin tocar el binario):** en `location /` del vhost `doors.azc.com.co` se agregó `proxy_set_header Accept-Encoding "";` — evita que el upstream gzipee, nginx recibe HTML plano con content-type correcto y lo gzipea él mismo (preservando type + `vary: Accept-Encoding`). Verificado: las 7 páginas del panel ahora devuelven `content-type: text/html; charset=utf-8` por nginx. Backup del conf en `nginx.ssl.conf.bak.gzipfix-*`.

**Lección:** `nosniff` + respuesta proxeada que pierde content-type = render texto plano. Si un upstream gzipea y no setea content-type, vaciar `Accept-Encoding` hacia el upstream y dejar que nginx comprima.

JS custom modificado: `DOOR_OPENER_BASE = "/door-opener"` (path relativo, mismo origin, mismo cert LE). El JS original llamaba `https://${window.location.hostname}:8445` que fallaba externamente.

**Importante:** después de cambios en `custom.js` los usuarios deben hard-refresh (Ctrl+Shift+R) para descartar cache del browser.

## Archivos clave

- `/etc/uhppoted/uhppoted.conf` — config principal con `bind.address=192.168.12.25`, `broadcast.address=192.168.15.255`, `listen.address=192.168.12.25:60001`
- `/etc/uhppoted/uhppoted-rest.conf` — REST API config
- `/etc/uhppoted/httpd/auth.json` — usuarios web
- `/etc/uhppoted/httpd/{ca.cert,uhppoted.cert,uhppoted.key}` — TLS
- `/var/uhppoted/httpd/system/` — controllers/cards/groups/doors/events/users/interfaces JSON
- `/opt/uhppoted-httpd-custom/` — UI custom branch (HTML/JS/CSS override)
- `/etc/systemd/system/uhppoted-{httpd,rest}.service`, `door-opener.service`

## Hestia firewall

LAN:
- `192.168.12.0/22 22 TCP` (LAN_SSH)
- `192.168.12.0/22 8543 TCP` (UHPPOTED_HTTPS)
- `192.168.12.0/22 8580 TCP` (UHPPOTED_HTTP)
- `192.168.12.0/22 8444 TCP` (UHPPOTED_REST)
- `192.168.12.0/22 8445 TCP` (DOOR_OPENER)
- `192.168.15.10 60001 UDP` (UHPPOTED_LISTEN para events del controlador)

WAN:
- `0.0.0.0/0 22 TCP` (SSH_WAN) — protegido por fail2ban-SSH
- `0.0.0.0/0 80,443 TCP` (web público para nginx, Hestia default)

## DNS público

`doors.azc.com.co` se gestiona en **uscloudlogin (ZAPI)** — donde está azc.com.co. NO en no-ip (eso es solo grupoazc.com).

## Backups de la migración

- `/root/backups/pre-uhppoted-migration-20260529-*` — configs originales antes de cambios
- `/root/backups/pre-ip-migration-20260529-103254/` — backup completo del server antes de la migración 192.168.0.206 → 192.168.12.25

## Sede Tequendama — controladores remotos vía uhppoted-tunnel (2026-06-02)

**4 controladores en red `192.168.14.x`** (otra sede, sin IP fija ni port-forward en su ISP): SN 223205300/.13, 225088590/.125, 423150802/.150, 425036574/.12. **Flag captura:** máscaras inconsistentes (225088590 tiene /22 `255.255.252.0`, los demás /24) — normalizar a la máscara real de la LAN + gateway antes de operar.

**Transporte elegido: uhppoted-tunnel (mTLS), relay dial-out desde Tequendama** → el Omada de Tequendama NO necesita nada; solo se abrió 1 puerto en el Omada del server. (Descartado: "wiedgan" era el software viejo N3000/Wiegand del mini PC, NO WireGuard — no hay túnel previo que reusar.)

**Relay = mini PC Windows de Tequendama** (el que corría el N3000, always-on, en 192.168.14.x).

**TÚNEL FUNCIONANDO end-to-end (2026-06-02):** los 4 controladores responden `get-device` desde el server central por el túnel. Detalles confirmados: 225088590=192.168.14.125 /22, 223205300=192.168.14.13 /24, 423150802=192.168.14.150 /24, 425036574=192.168.14.12 /24. **Todos con gateway 192.168.12.1** → por eso los /24 igual responden al relay (192.168.12.209, fuera de su /24): la respuesta sale por el gateway y vuelve dentro del /22. Normalizar máscaras a /22 es opcional, no bloqueante.

Comando de test (server): `uhppote-cli --bind 192.168.12.25:0 --broadcast 192.168.12.25:60010 --timeout 8s get-device <SN>`.

**Lado server (azc) — YA montado:**
- Service `uhppoted-tunnel-tequendama.service`: `--in udp/listen:192.168.12.25:60010 --out tls/server:0.0.0.0:60443 --ca-cert/cert/key + --client-auth`. **El listen UDP debe ser puerto dedicado `192.168.12.25:60010`, NO `0.0.0.0:60000`** — con 0.0.0.0:60000 captura el broadcast/discovery del propio uhppoted del server (a 192.168.15.255:60000, el server también es /22) y lo reenvía al relay → flood + rate-limit que ahoga las respuestas. Puerto dedicado lo aísla.
- **Release v0.9.0** (el binario se auto-reporta como `v0.8.12` al correr `version` — string embebido atrasado; no existe release v0.8.12, los tags van hasta v0.9.0). Ambos extremos del túnel deben usar el MISMO release (v0.9.0).
- **Relay Windows:** `--out udp/broadcast:255.255.255.255:60000` (broadcast limitado, independiente de máscara — un /22 hace que 192.168.14.255 NO sea broadcast válido y el paquete ni sale). Mini PC relay = 192.168.12.209 (NO .14.x), LAN Tequendama es 192.168.12.0/22.
- **Rate limit del túnel (crítico para bulk get-cards/get-acl/load-acl):** default 1 req/s burst 120 ahoga las operaciones masivas. Subido a **10/600**. **NO funciona como key top-level del TOML** — debe ir dentro de una **sección nombrada** referenciada con `--config archivo.toml#seccion`. Server usa `/etc/uhppoted/tunnel/tunnel.toml` sección `[tequendama]` con in/out/certs/client-auth/console/udp-timeout/rate-limit/rate-limit-burst, e invoca `--config .../tunnel.toml#tequendama` (el unit ya NO usa flags CLI sueltos). El relay Windows debe tener su PROPIO TOML con rate-limit 10/600 (cada extremo limita por separado; el cuello de botella es el menor). `--udp-timeout 5s` por la latencia del túnel.

**Reconciliación de tarjetas ANTES de publicar a Tequendama (load-acl BORRA las no listadas):** cosechar el ACL actual de las 5 placas (get-acl) → clasificar cada card (solo Palmetto / solo Tequendama-X / en varias) con su acceso por puerta → importar a `cards.json` como superconjunto con grupos que reproduzcan el acceso actual → primer publish da `updated:0 deleted:0` (no quita acceso). Recién ahí gestión central. Controladores en conf con `.address=192.168.12.25:60010`. get-cards format: `<card> <from> <to> Y N N N [PIN]`.

**Reliability bulk read sobre túnel:** requiere **rate-limit 100/5000 en AMBOS extremos** (server tunnel.toml + relay tunnel.toml) — con 10/600 las lecturas venían incompletas/variables. Con 100/5000 get-cards es estable y completo. Patrón: el PRIMER get-device/get-cards tras idle suele dar timeout (warmup ARP), reintentar. Cosecha en `/root/harvest/` (union_harvest.sh para placas flaky).

**COSECHA 2026-06-02 (datos confiables):**
- **Palmetto (222451671): 240 cards.**
- **Tequendama = sistema replicado de 227 cards únicas** en 4 placas: 225088590/.125=227, 425036574/.12=227, 223205300/.13=227, **423150802/.150=215 (le faltan 12, sin propias)**.
- **Cruce Palmetto↔Tequendama:** ambas sedes=115, solo Palmetto=125, solo Tequendama=112, **total personas únicas=352**.
- **Tequendama usa PIN de teclado** (campo extra en get-cards) → migración con `load-acl --with-pin` obligatorio o se borran PINs.
- Implicación: las 112 solo-Teq NO están en cards.json → importarlas antes de publicar o load-acl las borra. Las 115 compartidas necesitan grupos que abran puertas de ambas sedes.
- Certs mTLS en `/etc/uhppoted/tunnel/` (ca, server, client; SAN del server = doors.azc.com.co + 186.145.239.174 + 192.168.12.25; válidos 10 años).
- Firewall Hestia: `ACCEPT 0.0.0.0/0 60443 TCP` (TUNNEL_TEQ, regla 19).
- **Port-forward 60443/TCP → 192.168.12.25 ya abierto en Omada Palmetto** (el del server Mail+Puertas).
- Bundle para el relay en `/root/tequendama-relay.tar.gz` (ca.cert, client.cert, client.key + LEEME.txt con comandos).
- **Flag rate limit:** el túnel loguea `1 req/s, burst 120` por defecto → un `load-acl` de 240 cards puede tardar ~2 min. Afinar con `--config` TOML si molesta.

**Lado Windows (relay) — PENDIENTE (lo hace Koichi):**
- Bajar `uhppoted-tunnel.exe` v0.8.12 (GitHub releases), copiar los 3 certs.
- Cliente: `uhppoted-tunnel.exe --console --in tls/client:186.145.239.174:60443 --out udp/broadcast:192.168.14.255:60000 --ca-cert ca.cert --cert client.cert --key client.key`. Luego `daemonize` como servicio.

**Tras el túnel:** registrar los 4 controladores en uhppoted.conf con `.address` apuntando al listen local del túnel (`192.168.12.25:60000` / `127.0.0.1:60000`) — el protocolo es serial-addressed, el broadcast remoto entrega al correcto. Door labels `S2 ...`. Profiles con **mismos ids** que Sede 1. `publish`/load-acl ya empuja a ambas sedes.

## Registro de controladores en el panel + automatización (2026-06-02)

Las 4 placas de Tequendama quedaron registradas en el panel (`controllers.json` 0.2.2-0.2.5 + `doors.json` 0.3.5-0.3.16). Ahora el panel muestra **5 controladores / 16 puertas** (4 Palmetto + 12 Teq) → la asignación de grupos ya puede conceder puertas de ambas sedes. controllers.json `address` para Teq = `192.168.12.25:60010` (cosmético; el panel comunica vía uhppoted.conf — Palmetto tiene un address distinto en cada archivo y funciona, prueba de que usa conf). JSON owner root:root, service corre como root. Door usage real Teq: .13=P1,P2 · .125=P1,P2 · .150=P1-P4 · .12=P1-P4.

**Automatización `register-controller`** (`/usr/local/bin/register-controller`): un comando idempotente que para una placa hace conf address+protocol, valida con get-device (reintenta por warmup), door labels en conf, y registro en panel (controllers.json+doors.json con OIDs auto, backup, restart httpd). Uso:
```
register-controller <serial> "<nombre>" <doors:1,2|auto> [address-tunel] [prefix]
```
Auto-detect de puertas NO es fiable sobre el túnel (lecturas parciales) → pasar puertas explícitas. **Prerequisito manual por sede nueva** (no automatizable server-side): montar el relay Windows + listen del túnel en puerto dedicado distinto + abrir puerto en Omada. Una vez la placa responde get-device, `register-controller` hace el resto.

**httpd pollea Teq por el túnel cada 30s** → logs `WARN read udp i/o timeout` ocasionales (warmup/intermitente), no crítico; el panel puede mostrar Teq offline a ratos.

## Eventos multi-placa + salto de cursor (2026-06-02)

Al registrar las placas Teq, httpd empieza a traer eventos **desde el índice 1 (más viejo) hacia adelante** → por el túnel tarda eternidades en llegar a hoy, así que el tab de Eventos solo mostraba Palmetto (los Teq quedaban sepultados con timestamps viejos: 2017/2022/2025). **Relojes de los controladores OK** (get-time = hoy), o sea no era desfase; era el cursor. httpd reanuda desde el índice `index` máximo por device en `events.json`.

**Fix (salto de cursor):** script `/root/cursor_jump.py` — para cada placa Teq hace `get-events` (rango), trae los 25 eventos más recientes con `get-event <serial> <idx>`, los inyecta en `events.json` con OID `0.6.N` continuando el máximo + door-name resuelto, y borra los viejos de esa placa. Tras eso httpd reanuda desde el índice reciente → eventos nuevos aparecen solos. **Correr con httpd detenido** (stop → script → start). Si una placa da timeout en get-events (warmup), warmup con get-device y reintentar. Backups `events.json.bak.cursorjump2-*`. Formato get-event: `serial idx timestamp card door granted reason`. get-events: `serial first last current`.

## Poll de httpd a placas tuneladas — warmup timeouts + keepalive (2026-06-02)

Al registrar las 4 placas Teq en el panel, httpd las pollea cada 30s por el túnel. **Problema:** entre polls hay 30s de idle → el primer paquete a cada placa Teq se cae (warmup ARP/path del túnel), genera flood de `WARN read udp ...60010 i/o timeout`. RTT del túnel caliente es 0.04s (rapidísimo); el problema es solo el arranque en frío. Esos warmups, sumados a correr CLI pesado (cursor_jump, 100 get-event) **concurrente** con los polls, atascaron el loop de httpd y dejaron de actualizarse los eventos (incluso Palmetto, que es directo, se starvó). Restart de httpd lo destraba.

**Fix permanente:** servicio `teq-keepalive.service` (`/usr/local/bin/teq-keepalive.sh`) — loop que hace `get-device` a las 4 placas Teq cada 15s, manteniendo el path caliente. Resultado: timeouts de httpd a 60010 cayeron de ~14/60s a **0**. Eventos de las 5 placas actualizan confiable (~30s). **Regla:** no correr ops CLI pesadas por el túnel concurrente con httpd; si los eventos se atascan, restart httpd. httpd default `udp.timeout`=2.5s (no en conf); subirlo NO ayuda (el warmup se DROPEA, no es lento) — el keepalive es la solución.

## httpd NO puede pollear multi-controlador por UN túnel — Opción B (2026-06-02)

**Causa raíz (arquitectural):** uhppoted-tunnel colapsa las 4 placas Teq en UNA dirección (`192.168.12.25:60010`). httpd las pollea **concurrente** y no puede demuxar las respuestas (todas vienen de la misma addr) → las cruza (`WARN invalid controller ID - expected:X, got:Y`) → el loop de poll se cuelga y arrastra a Palmetto. `uhppote-cli` no sufre (1 request a la vez). **Esto pasaría en cada sede nueva por túnel uhppoted.** El fix correcto sería WireGuard subnet-router (IPs reales distintas) o 1 puerto de túnel por controlador — ambos pesados/multi-instancia.

**Decisión: Opción B (poller secuencial dedicado).**
- **Estabilización:** se quitaron las 4 placas Teq de `controllers.json` → httpd **solo pollea Palmetto** (tiempo real, 0 cruces). Se **conservan** en `doors.json` (16 puertas, grupos siguen funcionando) y en `uhppoted.conf` (CLI/poller las alcanza). Backups `controllers.json.bak.poll-*`.
- **Por qué no inyectar en el tab nativo:** httpd tiene `events.json` en memoria; inyecciones externas las pisa en su próximo persist salvo restart. Por eso Teq va en tab aparte.
- **Poller:** `teq-events-poller.service` (`/usr/local/bin/teq-events-poller`) — secuencial, cada 60s hace `get-events` (con 3 reintentos por warmup) + `get-event` incremental por cursor, escribe `/var/uhppoted/teq-events.json` (events+cursor por placa, retiene 3000). NO toca httpd. .13 (223205300, v6.62) es flaky en frío → los reintentos lo cubren.
- **Display:** endpoint `GET /api/teq-events` en schedule-manager + tab **"Eventos Teq"** en `/schedules/index.html` (tabla Hora/Placa/Puerta/Tarjeta/Acceso, auto-refresh 30s). El tab nativo de Eventos queda para Palmetto.
- **`teq-keepalive.service`** sigue (pinguea las 4 cada 15s) — mantiene el path caliente para el poller y on-demand CLI.

## Seguridad ACL — NO publicar hasta reconciliar (regla dura)

Asignar grupos→puertas en el panel NO toca los controladores (solo edita groups.json); nada cambia hasta un publish/load-acl. **Antes de cualquier publish: `compare-acl <tsv>`** = dry-run que muestra add/delete/update por controlador sin tocar nada. Publicar el cards.json actual borraría las 112 solo-Teq → quedarían sin entrar. **Algunas puertas de Teq usan el MISMO lector para entrada y salida** → revocar acceso ahí deja a la persona ENCERRADA (no solo afuera). Por eso reconciliación perfecta + compare-acl obligatorios antes de publicar a Tequendama.

## Pendientes (próximas sesiones)

- ~~Bug System (controllers.html) texto plano~~ RESUELTO 2026-06-02 (gzip upstream borra content-type → fix `proxy_set_header Accept-Encoding ""` en nginx; ver sección dedicada arriba).
- **Time profiles reales:** definir los 3-5 horarios concretos por rol (Empleados, Directivos, Servicios Generales, etc.) y aplicar masivamente con la UI Schedules.
- **Cambiar password admin de uhppoted** (sigue en default `admin / uhppoted`)
- **NAT hairpin / DNS interno**: clientes en LAN de la oficina del server tienen problema accediendo `doors.azc.com.co` por NAT loopback. Workaround temporal: `/etc/hosts → 192.168.12.25 doors.azc.com.co` por máquina. Fix definitivo: habilitar hairpin en Omada, o configurar zona DNS local en BIND9 del server (named ya corre).
- **Auth en `/schedules/api/`**: actualmente sin auth — cualquiera con acceso al dominio puede modificar profiles. Agregar basic auth o validar cookie de uhppoted-httpd.
- Configurar uhppoted-tunnel (binario instalado pero sin uso)
- Vaciar cola exim del server azc (4228 mensajes frozen, no urgente)

## Lecciones aprendidas

- **uhppoted-httpd no recibe eventos por UDP** — polling cada 30s a `httpd.system.refresh`. UDP listener es de uhppoted-rest.
- **Hestia ya usa 8080 y 8443** para Apache backend. uhppoted-httpd debe ir a otros puertos (8543/8580 elegidos).
- **`enforce_subdomain_ownership=yes`** en Hestia: subdominio debe ir al mismo user que el dominio padre. `doors.azc.com.co` → user `azcweb` (mismo que azc.com.co).
- **CA cert + key debe ser pareja válida**: la CA vieja del project tenía cert/key mismatch — regeneré ambas.
- **Sintaxis set-listener:** `set-listener <serial> <ip:port>` (con colon, no space).
- **`sudo rsync` falla con StrictHostKeyChecking** porque sudo cambia HOME a /root sin known_hosts. Usar `sudo tar -czf - <path> | ssh root@dest 'tar -xzf - -C /'` para transferir como root preservando perms.
- **Hestia `v-add-letsencrypt-domain` fallaba con `Let's Encrypt finalize bad status 403 (orderNotReady, status invalid)`** aunque el webroot servía correctamente desde Internet. `certbot certonly --webroot` directo funcionó al primer intento. Para LE en Hestia, si v-add falla repetidamente, fallback a certbot directo y copiar manualmente a `/home/<user>/conf/web/<domain>/ssl/{crt,key,ca,pem}` (cert.pem→crt, chain.pem→ca, privkey.pem→key, fullchain.pem→pem).
- **DNS de azc.com.co está en `supremedns.com`** (NS de uscloudlogin/ZAPI), no en no-ip. Cuando agregás A record, TTL=1800 (30 min). LE puede usar resolvers con cache viejo y validation falla — esperar propagation o usar certbot directo.
- **`v-add-web-domain` requiere user dueño del dominio padre** por `ENFORCE_SUBDOMAIN_OWNERSHIP=yes` en hestia.conf. `doors.azc.com.co` debe ir bajo `azcweb` (mismo user que `azc.com.co`).
- **systemctl reload nginx no siempre carga cert nuevo** — usar `systemctl restart nginx` después de cambiar certs SSL para forzar workers nuevos.

## Reglas duras

- ACL sync en httpd es destructivo: sobrescribe permisos del controlador con lo que tenga el servidor. Siempre backup `/tmp/raw_cards.txt` antes.
- UDP sobre internet pierde paquetes — door-opener tiene retry x3 por esto.
- Datos demo del starter kit (Hogwarts) deben eliminarse antes de operar.
