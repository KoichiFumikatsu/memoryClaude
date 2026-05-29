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

- CA self-signed regenerada el 2026-05-29 en `/etc/uhppoted/httpd/ca.cert` (válida 10 años)
- Server cert en `/etc/uhppoted/httpd/uhppoted.cert` CN=`doors.azc.com.co`, SAN: `IP:192.168.12.25, IP:186.145.239.174, IP:185.42.23.162, IP:127.0.0.1, DNS:doors.azc.com.co`
- Cert para nginx vhost en `/home/azcweb/conf/web/doors.azc.com.co/ssl/` — autofirmado por ahora, **PENDIENTE LE cuando DNS propague**

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

## Lecciones aprendidas

- **uhppoted-httpd no recibe eventos por UDP** — polling cada 30s a `httpd.system.refresh`. UDP listener es de uhppoted-rest.
- **Hestia ya usa 8080 y 8443** para Apache backend. uhppoted-httpd debe ir a otros puertos (8543/8580 elegidos).
- **`enforce_subdomain_ownership=yes`** en Hestia: subdominio debe ir al mismo user que el dominio padre. `doors.azc.com.co` → user `azcweb` (mismo que azc.com.co).
- **CA cert + key debe ser pareja válida**: la CA vieja del project tenía cert/key mismatch — regeneré ambas.
- **Sintaxis set-listener:** `set-listener <serial> <ip:port>` (con colon, no space).
- **`sudo rsync` falla con StrictHostKeyChecking** porque sudo cambia HOME a /root sin known_hosts. Usar `sudo tar -czf - <path> | ssh root@dest 'tar -xzf - -C /'` para transferir como root preservando perms.

## Reglas duras

- ACL sync en httpd es destructivo: sobrescribe permisos del controlador con lo que tenga el servidor. Siempre backup `/tmp/raw_cards.txt` antes.
- UDP sobre internet pierde paquetes — door-opener tiene retry x3 por esto.
- Datos demo del starter kit (Hogwarts) deben eliminarse antes de operar.
