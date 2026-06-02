---
name: project_uhppoted
description: uhppoted access control system — migrado a servidor central azc.com.co (192.168.12.25) el 2026-05-29. Subdominio doors.azc.com.co
type: project
---
# Estado actual (2026-05-29)

**MIGRADO al servidor central:** `azc.com.co` (LAN `192.168.12.25`, NAT `186.145.239.174`). Fumilinux (192.168.12.168) ya **NO** corre uhppoted — services stopped + disabled.

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
