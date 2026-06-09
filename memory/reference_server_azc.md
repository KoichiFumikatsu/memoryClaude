
## ⚠️ DOS servers distintos (no confundir) + acceso (2026-06-09)

- **Este (azc.com.co / uhppoted)** = LAN `192.168.12.25`, NAT **`186.145.239.174`**. Desde fuera de la LAN (Fumilinux remoto) la LAN .25:22 da **"refused"** → usar **`ssh root@186.145.239.174`** (regla SSH_WAN). Ubuntu 20.04, Hestia 1.6.8, 8GB RAM (~4.7 avail), 1.2TB disco libre, PHP 7.3/7.4/8.0(+8.3 sin fpm), php8.0-fpm activo, MariaDB 10.6.5.
- **`cloud.azclegal.com`** = box DISTINTO: `185.42.22.239` / LAN `192.168.12.234`. Hestia 1.9.6, **94GB RAM, 7.3TB (2.2TB libre)**, PHP 8.0–8.4, MariaDB 11.4, corre **Nextcloud `mi.azclegal.com` (user nube)**. El alias **`azc-root`** en `~/.ssh/config` apunta a ESTE (.234), NO al de uhppoted.
- Users Hestia en .25: admin, **azclegal** (mi./nube.azclegal.com), azcweb (azc.com.co, doors.azc.com.co), bilinguelaw, grupoazc, mailazc, mailer.

## DEV AZCKeeper `devkeep.azclegal.com` en .25 — plan (2026-06-09, NO ejecutado)

Reemplaza `projects.k.azclegal.com`. Pasos:
1. DNS (Koichi): A `devkeep.azclegal.com` → 186.145.239.174 (80/443 ya forwardeados; doors es público).
2. Vhost Hestia bajo user **`azclegal`** (enforce_subdomain_ownership on), template **nginx+php8.0** (sin Apache), docroot → `AZCKeeper_Client/Web/public`.
3. BD MariaDB local `azckeeper_dev` (+user); esquema desde `Web/migrations/*.sql` o dump schema-only keeper_*.
4. Deploy backend vía git desde **DevLinux** (solo `AZCKeeper_Client/Web`).
5. `.env` dev (APP_ENV=dev, BD local). Pendiente LEGACY_DB_* (employee): host externo RO o snapshot local.
6. LE con `certbot certonly --webroot` (v-add-letsencrypt falla acá: 403 orderNotReady).
7. ApiBaseUrl DEV del cliente → `https://devkeep.azclegal.com/...`; retirar projects.k tras validar.
Caveat: .25 = control de puertas en prod, 8GB → DEV liviano, sin jobs pesados (backfill window_episode / load-tests screenshots → cloud .234 o ventana).
