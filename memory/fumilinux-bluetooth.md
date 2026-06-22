---
name: project_bluetooth_fumilinux
description: BT Fumilinux se cae con CUALQUIER dispositivo por Hardware error 0x0c del Intel 8087:0aaa; fix = desactivar USB autosuspend vía udev
metadata: 
  node_type: memory
  type: project
  originSessionId: 19d585a9-64f5-472c-8190-30d09a66ce38
---

Bluetooth de Fumilinux se desconectaba con **cualquier** dispositivo (no era el mouse, como se creía antes — se reprodujo igual con Logi M196 y Satechi M1).

**Causa raíz (2026-06-22):** controlador **Intel 9460/9560 Jefferson Peak, USB 8087:0aaa** lanza `Bluetooth: hci0: Hardware error 0x0c` recurrente. Cada error fuerza reconexión del HID (se ve como dispositivo "que se cae"). Disparador: **USB autosuspend** del adaptador (`power/control = auto`, 2000 ms).

**Fix aplicado y persistido:**
- Regla udev: `/etc/udev/rules.d/50-bluetooth-no-autosuspend.rules` → fuerza `power/control = on` para `idVendor 8087 / idProduct 0aaa`.
- En caliente se aplicó con `echo on | sudo tee .../power/control` + reset del controlador (`bluetoothctl power off/on`).

**Verificar si vuelve:** `sudo dmesg | grep -i "hci0: Hardware error"` → vacío = OK.

**Escalado si reaparece** (0x0c en Intel también es coexistencia WiFi/BT y bugs de firmware): actualizar `linux-firmware` (estaba `20240318.git3b128b60-0ubuntu2.27`) o forzar reset de firmware vía `btintel`/recarga de `btusb`.
