---
name: reference_server_azc_cloud
description: Server cloud.azclegal.com (185.42.22.239 / LAN 192.168.12.234) — el "servidor de nube" de AZC, hospeda Nextcloud (user nube:nube). Hestia Ubuntu 22.04, ~500 usuarios. Multi-WAN Omada ER707-M2 con trampa de policy routing.
type: reference
---

# Server cloud.azclegal.com — Stack y red

**Distinto del server `azc.com.co`** (192.168.12.25 / 186.145.239.174) en `server-azc-stack.md`. Este es el cloud de producción con ~500 usuarios.

## Identidad

- Hostname `cloud.azclegal.com`
- IP LAN: `192.168.12.234/24` en `enp0s25`, NIC `enp5s0` DOWN (sin cable)
- IP pública (NAT vía Omada): `185.42.22.239` (= DNS `mi.azclegal.com`)
- Gateway LAN: `192.168.12.1` (Omada ER707-M2 multi-WAN)
- OS: Ubuntu 22.04.5 LTS · Kernel 5.15.0-179
- HW: i5+ class, 94 GB RAM, 7.3 TB en `/dev/md1` (~70% usado)
- HestiaCP (`INTERFACE='enp0s25'`, `NAT='185.42.22.239'` en `/usr/local/hestia/data/ips/`)

## Stack (`systemctl is-active` confirmado)

- nginx (`192.168.12.234:80/443`) — frontend
- apache2 (`192.168.12.234:8080`) — backend
- hestia (panel `:8083`)
- exim4 (25/465/587), dovecot (110/143/993/995), vsftpd (21), BIND9 named (53), MariaDB (3306 local)
- fail2ban con 7 jaulas (`ssh-iptables`, `dovecot-iptables`, `exim-iptables`, `hestia-iptables`, `phpmyadmin-auth`, `recidive`, `vsftpd-iptables`)
- **Nextcloud** — este es el server que lo hospeda (el "servidor de nube" de AZC). Ver sección dedicada abajo. Regla crítica: corre como user **`nube:nube`**, NUNCA `chown -R www-data`.

## SSH

Pubkey de Fumilinux instalada en `root@185.42.22.239`. Acceso configurado en `~/.ssh/config` (Fumilinux):

```
Host torre1
    HostName 100.67.216.43
    User kelsielinux

Host azc-root
    HostName 185.42.22.239
    User root
    ProxyJump torre1
```

Uso: `ssh azc-root`. ProxyJump por Torre 1 porque la IP pública de Fumilinux quedó baneada por fail2ban en intentos iniciales (su ban expira; Torre 1 = bypass permanente).

## Trampa CRÍTICA: routing asimétrico multi-WAN

El Omada del datacenter (ER707-M2) tiene 2+ WANs:
- **WAN1** → IP pública `185.42.22.239` (la del dominio, donde llegan los 500 usuarios)
- **WAN2** → IP pública `200.29.106.240` (default outbound)

**Sin policy routing**, el server sale por WAN2 → respuestas a usuarios externos llegan con src=`200.29.106.240` cuando el cliente envió SYN a `185.42.22.239` → cliente descarta SYN-ACK como "unsolicited" → handshake nunca completa → timeout en HTTPS/SSH/etc para usuarios externos.

**Fix obligatorio**: Policy Route en Omada:
- Source IP: `192.168.12.234/32`
- Destination: Any
- Protocol/Port: All
- Target WAN: WAN1 (la de `185.42.22.239`)
- Effective: Always

Validación rápida (debe dar `185.42.22.239`):
```
ssh azc-root "curl -s https://api.ipify.org"
```

Si da otra IP, el policy está roto o desactivado → usuarios externos NO entrarán correctamente.

## Trampa secundaria: sesiones cacheadas en Omada

Cambios de policy routing o Load Balancing NO se aplican a sesiones existentes. Tras cambios, hacer **reboot del ER707-M2** (1-2 min downtime) para limpiar conntrack. Sin reboot, el síntoma es que la mitad de los puertos funcionan y la otra mitad no, por puerto+source-IP hash.

## ISP outbound de Fumilinux filtra :443 a 185.42.22.x

Desde la oficina Fumilinux específicamente, TCP/443 a `185.42.22.239` no sale al internet (traceroute TCP muere en hop 3, en el edge del ISP de la oficina). Otros HTTPS sí funcionan, ICMP a esa IP sí pasa. Workaround: usar `ssh azc-root` (tunela por Torre 1) o probar desde datos móviles. **Esto NO afecta a los 500 usuarios** — es solo Fumilinux ↔ ese destino específico.

## Convención multi-sede

Todas las sedes usan LAN scheme `192.168.12-15.x` con Omada como controller. Eso significa que la subred `192.168.12.x` aparece en MÚLTIPLES sedes — no asumir que dos hosts en `192.168.12.x` están en la misma red física.

## Lecciones operacionales

- Antes de tocar firewall/Omada en producción: confirmar la WAN que sostiene la IP pública del dominio y NO romper su path de retorno.
- "Firewall inactive" en Omada **no desactiva** Session Limit, Attack Defense, Bandwidth Control ni Policy Routing — son menús independientes.
- ER707-M2 puede manejar ~150K sesiones — la capacidad rara vez es el problema; suele ser config.

## Nextcloud — `mi.azclegal.com` (verificado on-box 2026-06-12)

- **Versión** Nextcloud 31.0.7. Docroot `/home/nube/web/mi.azclegal.com/public_html/`. Dominio real **`mi.azclegal.com`** (NO `nube.azc.com.co` — esa atribución heredada era falsa).
- **Cadena web**: nginx `:443` → apache `:8443` (mod_remoteip, log con IP real) → **php-fpm 8.2** socket `/run/php/php8.2-fpm-mi.azclegal.com.sock`, pool `mi.azclegal.com`, user/group `nube:nube`. Ojo: el **CLI `php` es 8.3**, pero el **web corre en 8.2** (el pool `cloud.azclegal.com.conf` de php8.3 es de OTRO vhost, irrelevante).
- **occ**: `sudo -u nube php /home/nube/web/mi.azclegal.com/public_html/occ ...` (corre como root da warning; usar el user nube). Salida ensucia con warning PCNTL → filtrar.
- **BD** MariaDB `nube_bd` (localhost). `max_connections=200`. **Cache**: APCu local + **Redis para locking** (`memcache.locking=\OC\Memcache\Redis`).
- **Acceso DB read-only sin exponer secretos**: `mysql -N -e "SELECT ... FROM nube_bd.oc_*"` funciona por socket-root (NO hacer `cat config.php` ni `occ config:list system` → vuelcan dbpassword/secret/passwordsalt).
- **Logs**: access/error reales en `/var/log/apache2/domains/mi.azclegal.com.{log,error.log}` (nginx/domains está vacío). nextcloud.log JSON en `<docroot>/data/nextcloud.log`.
- **Nota seguridad pendiente**: `datadirectory` está DENTRO de `public_html` (`.../public_html/data`) — riesgo de exposición, tema aparte sin resolver.

### Incidente caídas de cliente / "bloqueo de IP" — RESUELTO 2026-06-12

**Síntoma**: clientes de escritorio (mirall) de varias sedes se caían "después de que se conectan varios usuarios desde esta IP". Omada e ISP descartados (no escalan con nº de usuarios). Causa = config del server, dos factores que se retroalimentan:

- **(A) 504 / desconexiones**: pool php8.2 estaba en `pm=ondemand`, **`pm.max_children=8`** en un server de **72 cores / 94 GB**. 8 workers para todo el Nextcloud multi-sede → al pasar la concurrencia de 8, las peticiones se encolaban en el socket FPM → apache `Timeout 30` → **504** → el cliente entra en backoff. Load bajo (2.4) NO lo reflejaba: el cuello era el techo configurado, no CPU.
- **(B) 429 / "bloqueo de IP"**: la protección brute-force de Nextcloud (`OCA\DAV\Connector\Sabre\Exception\TooManyRequests`) cuenta intentos de auth **por IP de origen**. Como cada sede sale por **una NAT**, los intentos de N clientes se suman a una sola IP → cruza el umbral → 429 a TODA la sede. Confirmado en nextcloud.log: 186.81.100.143 concentraba 435 de 468 `429`.

**Fix aplicado** (read-only scan primero; cambios con backup en `/root/backups/nextcloud-fpm-<ts>/`):
1. Pool `/etc/php/8.2/fpm/pool.d/mi.azclegal.com.conf`: `pm=dynamic`, `max_children=120` (elegido para quedar **bajo `max_connections=200` de MariaDB** y no cambiar 504 por 500), `start_servers=20`, `min_spare=10`, `max_spare=30`. `php-fpm8.2 -t` + `systemctl reload php8.2-fpm` (graceful, sin downtime). RSS real ~110 MB/worker → pico 120×110≈13 GB de 90 libres.
2. Allowlist brute-force = app config **appid `bruteForce`**, keys **`whitelist_<N>`** = CIDR (lo lee `OC\Security\Ip\BruteforceAllowList::isBypassListed`; gestionado por app `bruteforcesettings` 4.0.0 / Settings→Seguridad). Comando: `occ config:app:set bruteForce whitelist_N --value="<ip>/32"`. Agregadas las 10 NAT de sede (mirall): 186.81.100.143, 186.113.156.228, 200.29.100.2, 190.65.139.32, 181.58.39.142, 200.29.106.240, 186.145.239.174, 190.249.138.88, 190.249.168.8, 186.168.98.201 (ya existía `whitelist_1=200.29.101.5/24`). **Son IP de ISP: si una sede rota de IP, re-agregar (o usar /24).**
3. `occ config:system:set trusted_proxies 0/1 --value 127.0.0.1 / 192.168.12.234` + `overwriteprotocol=https` (antes vacíos; la IP real funcionaba solo por mod_remoteip de apache).

**Verificación runtime**: status page del pool (`pm.status_path=/status`) → `process manager: dynamic`, `max children reached: 0`, `listen queue: 0`. Pendiente: observar 429/504 bajo carga real para confirmar que desaparecen.
