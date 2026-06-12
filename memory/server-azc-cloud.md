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
- **Nextcloud** — este es el server que lo hospeda (confirmado por Koichi 2026-06-12). Es el "servidor de nube" de AZC. Regla crítica: corre como user **`nube:nube`**, NUNCA `chown -R www-data` (rompió el server una vez). **Pendiente verificar on-box**: ruta exacta del docroot, dominio (¿`nube.azc.com.co`? esa atribución venía mal-copiada del server `.25`), versión de Nextcloud y BD que usa. No asumir la ruta `/home/grupoazc/web/...` — era del `.25`.

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
