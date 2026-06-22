---
name: project_bluetooth_fumilinux
description: BT interno Intel de Fumilinux (Vivobook X1404VA) es defectuoso (Hardware error 0x0c); resuelto con dongle Realtek externo + Intel desactivado por udev
metadata: 
  node_type: memory
  type: project
  originSessionId: 19d585a9-64f5-472c-8190-30d09a66ce38
---

El Bluetooth de **Fumilinux (ASUS Vivobook X1404VA)** se desconectaba con **cualquier** dispositivo (no era el mouse — se reprodujo igual con Logi M196 y Satechi M1).

**Causa raíz (2026-06-22):** el módulo BT interno **Intel 9460/9560 (USB 8087:0aaa, firmware ibt-0040-1020 v151-5.24)** lanza `Bluetooth: hciX: Hardware error 0x0c` + `Exception info` cada ~2-3 min. Arranca limpio y crashea solo en operación. Firmware crash; el kernel se recupera reseteando el chip pero deja freezes.

**Descartado por diagnóstico (NO repetir estas vías):**
- USB autosuspend → desactivado, siguió fallando.
- Coexistencia WiFi/BT (`iwlwifi bt_coex_active=0`) → siguió fallando. (WiFi ya en 5 GHz.)
- Firmware desactualizado → el `ibt-0040-1020.sfi` instalado YA es idéntico al upstream de linux-firmware (mismo SHA256). No hay versión más nueva. Vía agotada.
- Conclusión: bug de firmware Intel sin parche o módulo M.2 defectuoso.

**Solución aplicada (2026-06-22):** dongle BT externo **Realtek RTL8761BU (TP-Link UB500, USB 2357:0604)** → aparece como controlador limpio (firmware rtl8761bu_fw.bin OK). El módulo Intel se desactiva permanentemente desautorizando su USB:
- `/etc/udev/rules.d/50-disable-intel-bt.rules`: `ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="8087", ATTR{idProduct}=="0aaa", ATTR{authorized}="0"` (no afecta al Realtek 2357:0604).

**Pendiente:** reemparejar los ratones al dongle (el pairing anterior era contra el adaptador Intel). Hacer modo-emparejamiento en el mouse + GNOME Settings > Bluetooth.

**Si el dongle no estuviera y hubiera que mitigar el Intel:** única opción restante era watchdog auto-reset de hci (no llegó a implementarse) o reasentar/RMA del módulo M.2 2230.
