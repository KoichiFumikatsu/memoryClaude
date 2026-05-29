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
- `location = /robots.txt` → archivo físico

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

## Hestia firewall (reglas LAN)

- `192.168.12.0/22 8543 TCP` (UHPPOTED_HTTPS)
- `192.168.12.0/22 8580 TCP` (UHPPOTED_HTTP)
- `192.168.12.0/22 8444 TCP` (UHPPOTED_REST)
- `192.168.12.0/22 8445 TCP` (DOOR_OPENER)
- `192.168.15.10 60001 UDP` (UHPPOTED_LISTEN para events del controlador)

## DNS público

`doors.azc.com.co` se gestiona en **uscloudlogin (ZAPI)** — donde está azc.com.co. NO en no-ip (eso es solo grupoazc.com).

## Backups de la migración

- `/root/backups/pre-uhppoted-migration-20260529-*` — configs originales antes de cambios
- `/root/backups/pre-ip-migration-20260529-103254/` — backup completo del server antes de la migración 192.168.0.206 → 192.168.12.25

## Pendientes (próximas sesiones)

- **Accesos por horario** (time profiles): los UT0311-L0x soportan slots horarios nativos. Definir grupos con horario (ej. Empleados 7am-7pm L-V). Falta diseño + implementación en panel.
- **Cambiar password admin de uhppoted** (sigue en default `admin / uhppoted`)
- **NAT hairpin / DNS interno**: clientes en LAN de la oficina del server tienen problema accediendo `doors.azc.com.co` por NAT loopback. Workaround temporal: `/etc/hosts → 192.168.12.25 doors.azc.com.co` por máquina. Fix definitivo: habilitar hairpin en Omada, o configurar zona DNS local en BIND9 del server (named ya corre).
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
