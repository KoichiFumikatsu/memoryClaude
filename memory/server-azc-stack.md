---
name: reference_server_azc
description: Server azc.com.co (192.168.12.25, NAT 186.145.239.174). Stack Ubuntu 20.04 + Hestia 1.6.8 + exim4 + dovecot + named + mariadb. Fixes legacy admin→hestiaweb aplicados 2026-05-29.
type: reference
---
# Server azc.com.co — Stack y acceso

## Hardware/OS

- Ubuntu 20.04.3 LTS · Kernel 5.15.0-139
- Intel i5-3470 (4 cores, 3.2 GHz Ivy Bridge)
- RAM 8 GB · Swap 2 GB
- Disco / = `/dev/sda4` 3.6 TB (67% usado)
- IP LAN `192.168.12.25/22` **estática** (NetworkManager `Conexión cableada 2`)
- IP pública NAT: `186.145.239.174`
- MAC eno1: `d4:c9:ef:f1:ad:e0` (reservado en Omada)
- Hostname: `azc.com.co`

## Acceso SSH

- root con clave pública (PermitRootLogin=prohibit-password, sin password)
- Clave pública de Fumilinux ya en `/root/.ssh/authorized_keys`
- iptables Hestia firewall: ACCEPT TCP 22 desde `192.168.12.0/22` (LAN)
- Comando: `ssh root@192.168.12.25`

## Stack instalado

- **Hestia 1.6.8** panel `:8083` (panel funcional tras fixes)
- **Apache 2.4** backend en `192.168.12.25:8080` (HTTP) y `:8443` (HTTPS)
- **nginx** frontend en `192.168.12.25:80` y `:443`
- **exim4** MTA: SMTP 25, 465, 587 (NO postfix aunque exista el paquete)
- **dovecot** IMAP/POP3: 110, 143, 993, 995
- **BIND9 named** DNS: 53
- **MariaDB 10.6.5** local
- **Roundcube 1.6.0** webmail (paquete Debian) en `/var/lib/roundcube/`
- **uhppoted 0.9.0** acceso (puertos 8543/8580/8444/8445 — ver memoria proyecto)

## Servicios web hospedados

| Dominio | DNS público | Apunta acá? |
|---|---|---|
| azc.com.co | 162.210.103.253 (uscloudlogin) | NO — público en otro hosting |
| bilinguelaw.com | 185.158.133.1 | NO |
| grupoazc.com | 181.60.79.75 (NAT viejo, muerto) | parcial |
| mail.grupoazc.com | 186.145.239.174 ✓ | SÍ (Roundcube) |
| webmail.grupoazc.com | 186.145.239.174 ✓ | SÍ (Roundcube) |
| mx.grupoazc.com | 186.145.239.174 ✓ | SÍ (exim MX) |
| doors.azc.com.co | (pendiente DNS uscloudlogin) | SÍ (uhppoted reverse proxy) |

## Fixes legacy admin → hestiaweb (aplicados 2026-05-29)

Hestia 1.5+ migró del usuario `admin` a `hestiaweb` para el panel. Este server tenía config vieja. Para que el panel sirva (no 502/500):

1. `/usr/local/hestia/nginx/conf/nginx.conf` → cambiar `user admin` a `user hestiaweb`
2. `chown -R hestiaweb:hestiaweb /usr/local/hestia/data/sessions /usr/local/hestia/nginx/fastcgi_temp /usr/local/hestia/web/fm/private`
3. `chmod 755 /usr/local/hestia/conf` (era `drwxr-x---`, hestiaweb no podía atravesar)
4. Crear `/etc/sudoers.d/hestiaweb` espejo del de admin (NOPASSWD para /usr/local/hestia/bin/*)

## DNS proveedores

- **`grupoazc.com`** y subdominios → no-ip.com (NS: ns1-ns5.no-ip.com)
- **`azc.com.co`** y subdominios → **uscloudlogin (ZAPI)** — donde está el hosting principal
- **`azclegal.com`** → desconocido

## Mail vhost / Roundcube

- vhost `mail.grupoazc.com` / `webmail.grupoazc.com` apunta a `/var/lib/roundcube/`
- Hestia template de mail vhost **no incluye SetHandler PHP** — agregar `apache2.conf_php` y `apache2.ssl.conf_php` como IncludeOptional con `SetHandler "proxy:unix:/run/php/php8.0-fpm-webmail.sock|fcgi://localhost"`
- Pool dedicado `/etc/php/8.0/fpm/pool.d/webmail.conf` corre como `www-data` con `open_basedir` que incluye `/var/lib/roundcube:/etc/roundcube` (los pools por dominio están scopeados al home del user y NO incluyen Roundcube)
- `chmod g+w /var/lib/roundcube/{temp,logs}` para que www-data pueda escribir

## SSL certs (mayoría expirados — los que no apuntan acá no se renuevan)

- `/etc/letsencrypt/live/grupoazc.com/` válido (renovado 2025-12-02, expira 2026-03-02) — solo este está en LE
- Resto en `/home/<user>/conf/web/<domain>/ssl/` self-signed o LE viejo
- Renovación requiere DNS apuntando al server + port 80 abierto

## Reglas críticas (no romper)

- **Postfix está `inactive` pero NO es problema** — exim4 es el MTA real
- **Nextcloud usa user `nube:nube`** en `/home/grupoazc/web/nube.azc.com.co/` — nunca chown www-data
- IP del server fue `192.168.0.206` históricamente (NAT viejo `181.60.79.75`). Migrada a `192.168.12.25` (NAT `186.145.239.174`) el 2026-05-29 vía `v-add-sys-ip` + `v-change-web-domain-ip` por dominio + `v-delete-sys-ip 192.168.0.206`
- Cola exim de 4228 mensajes frozen (cron locals → `noreply@azc.com.co`, `andres.zafra@azc.com.co`). Vaciar es seguro pero no urgente.

## Backups de cambios

`/root/backups/pre-ip-migration-20260529-103254/` contiene snapshots de `/usr/local/hestia/data`, `/home/*/conf`, `/etc/nginx`, `/etc/apache2`, `/etc/exim4`, `/etc/dovecot`, `/etc/bind`, iptables, sudoers admin, ownership state. Para revertir cambios de la migración.
