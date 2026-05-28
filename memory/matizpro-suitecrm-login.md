# Matiz CRM (matizpro.com / matiz.azc.com.co) — sesión cae al editar/reportar

**Fecha sesión:** 2026-05-27
**Estado:** diagnóstico completo, **NINGÚN cambio aplicado aún** en el servidor.
**Usuario en servidor:** `azcgroup@web13`
**Path SuiteCRM:** `/home/www/matiz.azc.com.co/`

## Síntoma reportado
- Login inicial funciona normal.
- Al ir a edición / reporte / consultoría → pide login otra vez.
- A veces aparecen 2 o 3 columnas de login simultáneas (dashlets recargando el HTML del form porque su AJAX vino corrupto).
- Diagnóstico de admin tarda ~5 min y luego re-pide login.

## Causa raíz confirmada

`error.log` tiene **6 319 líneas idénticas**:
```
PHP Warning: include(custom/metadata/af_puestos_af_employee_afMetaData.php):
failed to open stream: No such file or directory
```

El archivo `af_puestos_af_employee_afMetaData.php` no existe, pero 2 archivos `Extension/` lo incluyen. El fuente solo contiene el `include` roto — huérfano de una relación creada en Studio que quedó mal desinstalada.

El `Warning` contamina el JSON de cada AJAX → el JS no parsea → asume "no hay sesión" → renderiza el form de login en cada dashlet (2-3 columnas).

## Confirmaciones hechas en servidor

1. `ls` del metadata: no existe.
2. `grep -rn`: solo en 2 archivos `Extension/`.
3. `find custom -type f -name "*af_puestos_af_employee*"` devuelve los 4 huérfanos:
   - `custom/Extension/application/Ext/TableDictionary/af_puestos_af_employee_af.php`
   - `custom/Extension/modules/AF_Puestos/Ext/Vardefs/af_puestos_af_employee_AF_Puestos.php`
   - `custom/Extension/modules/AF_Employee/Ext/Vardefs/af_puestos_af_employee_AF_Employee.php`
4. `SELECT FROM relationships WHERE relationship_name LIKE 'af_puestos_af_employee%'`: **vacío**. Studio no regenerará.

## Ruido a IGNORAR

Bloque `MISMATCH WITH DATABASE` con `ALTER TABLE ... int(N)` que reaparece tras cada Quick Repair: falso positivo conocido en SuiteCRM 7 sobre MySQL 8 / MariaDB ≥ 10.2.7 (display width descartado). No relacionado con el login.

## Otros hallazgos no bloqueantes
- Relaciones duplicadas: `af_activos_fijos_users_1`, `af_puestos_users_1`, `gasto_egresos_accounts`.
- Falta `date.timezone="America/Bogota"` en php.ini.
- Módulo "FACTOR AUTH" activo — revisar solo si tras limpieza el problema persiste.

---

## PENDIENTE EJECUTAR

### Paso 1 — Limpieza huérfanos
```bash
cd /home/www/matiz.azc.com.co

mkdir -p ~/backup_af_puestos_af_employee_$(date +%Y%m%d)
find custom -type f -name "*af_puestos_af_employee*" -exec cp --parents {} ~/backup_af_puestos_af_employee_$(date +%Y%m%d)/ \;
ls -lR ~/backup_af_puestos_af_employee_$(date +%Y%m%d)/

find custom -type f -name "*af_puestos_af_employee*" -delete

rm -f custom/application/Ext/TableDictionary/tabledictionary.ext.php
rm -f custom/modules/AF_Puestos/Ext/Vardefs/vardefs.ext.php
rm -f custom/modules/AF_Employee/Ext/Vardefs/vardefs.ext.php

rm -rf cache/modules/* cache/themes/* cache/smarty/*
rm -rf cache/file_map.php cache/class_map.php cache/include/*

chown -R www-data:www-data custom/ cache/ 2>/dev/null || \
chown -R apache:apache custom/ cache/ 2>/dev/null || true
```

### Paso 2 — En el CRM como admin
1. Admin → Repair → Quick Repair and Rebuild (ignorar MISMATCH final).
2. Admin → Repair → Rebuild Relationships.
3. Logout + incógnito + login.

### Paso 3 — Verificar
```bash
> /var/log/.../matizpro.com-error.log
grep -c "af_puestos_af_employee" /var/log/.../matizpro.com-error.log
# Esperado: 0
```
Probar: editar registro, Reportes, Consultoría, diagnóstico admin.

## Evidencia local (Linux Kelsie)
- `~/Downloads/www.tar.gz`, `matizpro.com-access.log.txt`, `matizpro.com-error.log.txt`
- Extraído: `/tmp/matiz_diag/matiz.azc.com.co/`
