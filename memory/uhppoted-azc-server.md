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
