# Configuración Troncal SIP TIGO + UCM6308 detrás de TP-Link ER605

> Documentación completa de la sesión de configuración y troubleshooting.
> **Fecha:** 2026-06-24
> **Objetivo:** Conectar un troncal SIP de TIGO (entregado por una ONT Huawei vía enlace /30) a un PBX Grandstream UCM6308, usando un router TP-Link ER605, **sin romper** los troncales internacionales (DIDWW) ni el internet existente.

---

## 1. Inventario de hardware

| Equipo | Rol | Detalle |
|---|---|---|
| **TP-Link ER605 v?** | Router / Gateway (Omada Gigabit VPN Router) | Hace el enrutamiento y NAT entre LAN, internet y el enlace de TIGO |
| **Huawei OptiXstar HG8145X6-10** | ONT de TIGO (entrega el SIP Troncal) | Provee el enlace /30 hacia la red privada de voz de TIGO. **No da DHCP** en el puerto usado |
| **Grandstream UCM6308** | PBX / IP-PBX | IP fija `192.168.12.170`. Tiene puertos de red duales (no usados para TIGO en esta config) |
| **TP-Link Archer A7 v5.0** | Router de consumo (descartado para esto) | Solo tiene IPTV/VLAN bridge; NO sirve para el enlace enrutado de TIGO |

---

## 2. Topología de red

```
                 INTERNET (por WiFi/ISP)
                        │
                        │  IP pública WAN1: 185.42.22.239/16
                        │  Gateway: 185.42.22.238
                        ▼
   ┌─────────────────────────────────────────────┐
   │              TP-Link ER605                    │
   │                                               │
   │  WAN  (puerto internet) ── 185.42.22.239      │
   │       DNS: 8.8.8.8 / 1.1.1.1                  │
   │                                               │
   │  WAN/LAN1 (puerto TIGO) ── 172.24.249.66/30   │
   │       Gateway: 172.24.249.65                  │
   │       (DNS de TIGO removido — ver §6)         │
   │                                               │
   │  LAN ── 192.168.12.0/24                       │
   └───────────────┬───────────────────────────────┘
                   │ LAN
                   ▼
          UCM6308  192.168.12.170
                   │
            Extensiones / Teléfonos físicos
            App Wave (web, WebRTC)


   Enlace TIGO (/30 privado, NO da internet):
   ER605 WAN/LAN1 (172.24.249.66) ──► Gateway TIGO 172.24.249.65 ──► red privada TIGO ──► SIP Server 172.17.179.150
```

**Concepto clave:** El enlace de TIGO es un **punto a punto /30 enrutado** hacia la red privada de voz de TIGO. NO es bridge ni da internet. Solo enruta a los servidores SIP de TIGO. El internet sale por una red WiFi totalmente separada (WAN1).

---

## 3. Datos entregados por TIGO

```
IP:              172.24.249.66
Puerta/Gateway:  172.24.249.65
Máscara:         255.255.255.252   (/30)
DNS (sys server / SIP server): 172.17.179.150
Llamadas simultáneas: 4
TRONCAL 1:       300 924 8209
TRONCAL 2:       300 924 8210
SERIAL LLAMADA:  WO524702
Orden:           WO550178
```

**Autenticación del troncal:** por **IP** (no por registro/contraseña). TIGO identifica al cliente por su IP de origen autorizada (`172.24.249.66`). El "usuario" es el número del troncal (DID).

**Aclaración importante sobre el servidor SIP:**
- `172.24.249.65` = **gateway / siguiente salto** (otro extremo del /30).
- `172.17.179.150` = **servidor SIP real (SBC)** de TIGO. Está en **otra subred** (`172.17.x`), detrás del gateway. Por eso requiere **ruta estática** (no es red directamente conectada).

---

## 4. Datos de los troncales internacionales (DIDWW)

- Proveedor: **DIDWW** (Outbound Voice Trunks), autenticados por **IP**.
- IP pública autorizada en el portal DIDWW: incluye `185.42.22.239/32` y además `0.0.0.0/0` (cualquiera) → la IP **sí está autorizada**.
- Servidores de salida vistos: `nyc.us.out.didww.com` → `46.19.209.44` (señalización), media en `46.19.209.17`, `46.19.209.70`, `46.19.209.71`.
- Troncales en el UCM (tipo register SIP): `Interno_Bruto`, `AZC USA`, `MOAZC USA`, `Colombia`, `DIDww Incoming`, etc.
- **External Host del UCM:** `go.azclegal.com` → resuelve a `216.238.79.56` = **nube de Grandstream (gdms.cloud / RemoteConnect)**. Es para el setup de trabajo remoto, NO es la IP pública local.

---

## 5. Configuración final del ER605

### 5.1 WAN de internet (puerto `WAN`)
- Connection: Dynamic/según ISP
- IP: `185.42.22.239`, Gateway `185.42.22.238`
- **Primary DNS: `8.8.8.8` / Secondary DNS: `1.1.1.1`** ← CRÍTICO (ver §6)

### 5.2 WAN de TIGO (puerto `WAN/LAN1`)
- Connection Type: **Static IP**
- IP Address: `172.24.249.66`
- Subnet Mask: `255.255.255.252`
- Default Gateway: `172.24.249.65`
- **Primary DNS: en blanco** (TIGO es por IP; su DNS rompía la resolución pública — ver §6)
- MTU: `1500`, VLAN: sin activar

### 5.3 Ruta estática (Routing → Static Route)
| Nombre | Destination | Mask | Next Hop | Interface |
|---|---|---|---|---|
| SIP_SERVER | `172.17.179.0` | `255.255.255.0` | **`172.24.249.65`** | WAN/LAN1 |

> ⚠️ **BUG ENCONTRADO Y CORREGIDO:** el Next Hop estaba escrito mal como `172.17.249.65` (typo: "17" en vez de "24"). Eso impedía que el tráfico de la LAN llegara a TIGO. El correcto es **`172.24.249.65`** (el gateway del /30).

### 5.4 Load Balancing (Transmission → Load Balancing)
- "Enable Load Balancing": **desactivado**
- "Enable Application Optimized Routing": **desactivado**
- "Enable Bandwidth Based Balance Routing": **desactivado**

### 5.5 Link Backup (Transmission → Link Backup)
- **Desactivado.**
> ⚠️ Probamos Link Backup (Primary=WAN, Backup=WAN/LAN1) pero **dejó el enlace de TIGO en standby**, y el ER605 **no reenvía tráfico de la LAN por una WAN en standby** → el UCM no alcanzaba TIGO. Por eso quedó desactivado.

### 5.6 NAT-DMZ (Transmission → NAT → NAT-DMZ)
| Nombre | Interface | Host IP | Status |
|---|---|---|---|
| NATTIGO | WAN/LAN1 | `192.168.12.170` | Enabled |

> Reenvía las llamadas entrantes de TIGO al UCM. En el enlace privado de TIGO la DMZ es segura.
> Para el **internacional** se agregó reenvío/DMZ en **WAN1** hacia `192.168.12.170` (port-forward UDP 5060 + RTP 10000-20000, o DMZ temporal de prueba) — esto fue lo que arregló el audio internacional.

### 5.7 ALG (Transmission → NAT → ALG)
- **SIP ALG: DESACTIVADO** (línea base actual).
- FTP/H.323/PPTP/IPsec ALG: activados (default).
> Ver §8: el SIP ALG genera un conflicto irresoluble entre TIGO e internacional.

---

## 6. La saga del DNS (causa de mucho tiempo perdido)

**Síntoma:** al quitar el DNS de TIGO se "caía el internet"; al ponerlo, se caían los troncales internacionales.

**Causa raíz:** la **WAN de internet (WiFi) NO entregaba un DNS público usable**. El único DNS configurado era el de TIGO (`172.17.179.150`), que solo resuelve la red interna de TIGO, no dominios públicos.
- Con el DNS de TIGO puesto → los troncales internacionales (que registran por **nombre de dominio**) no resolvían → "unreachable".
- Sin el DNS de TIGO → no había ningún DNS → "no internet".

**Solución definitiva:**
1. Poner **DNS público fijo `8.8.8.8` / `1.1.1.1` en la WAN de internet (WAN1)**.
2. **Quitar** el DNS `172.17.179.150` de la WAN de TIGO (no se necesita, TIGO es por IP).

Resultado: internet resuelve siempre por Google/Cloudflare, independiente de TIGO. Verificado con `ping google.com` desde el UCM (resuelve a `108.177.11.139` / `172.217.28.110`).

---

## 7. Configuración del UCM6308

### 7.1 NAT (PBX Settings → SIP Settings → NAT)
- **External Host:** `go.azclegal.com` ← **NO TOCAR** (es la cara de RemoteConnect / troncales internacionales).
- Use IP address in SDP: ✓
- Get External IP via STUN: ✓
- External UDP/TCP Port: 5060 / TLS 5061
- **Local Network Address (lista):**
  - `192.168.12.170 / 24` (preexistente)
  - `172.17.179.0 / 24` (agregado para TIGO)

> ⚠️ **Efecto secundario clave:** agregar `172.17.179.0/24` a Local Network Address hace que el UCM trate a TIGO como "local" y **anuncie su IP privada `192.168.12.170`** en el SDP hacia TIGO. Esto es la causa del audio en una sola vía de TIGO (ver §8).

### 7.2 Troncal TIGO (VoIP Trunks)
- Tipo: **Peer SIP Trunk**
- Host Name: `172.17.179.150`  (el servidor SIP, NO el gateway)
- Transport: UDP, puerto 5060
- Register: **No** (por IP)
- CallerID/DID: `3009248209` y `3009248210`, 4 canales
- Códecs (Advanced → Codec Preference): **PCMU, PCMA** arriba, luego GSM, G.726, G.729, iLBC ✓ (correctos)
- Media Traversal Method: None

### 7.3 Troncales internacionales (DIDWW)
- Tipo: register SIP. El "heartbeat/qualify" viene **forzado por defecto** en register SIP (no se puede desactivar fácil).
- Los servidores *out* de DIDWW a veces no responden OPTIONS → pueden aparecer "unreachable" aunque funcionen.

---

## 8. Diagnóstico del audio (análisis de capturas .pcap)

Se capturó tráfico en el UCM (Ethernet Capture, interface Any, filtro vacío) durante llamadas reales y se analizó con tcpdump.

### 8.1 Llamada INTERNACIONAL (DIDWW) — RESUELTA ✅
- UCM anuncia en SDP: `c=IN IP4 185.42.22.239` (IP pública correcta).
- UCM **envía** RTP a `46.19.209.17:17152` (puerto correcto).
- **Primera captura:** DIDWW devolvía solo **13 paquetes** (solo RTCP) → audio en una sola vía.
- **Tras aplicar DMZ/forwarding en WAN1:** DIDWW devuelve **1663 paquetes RTP** → **audio en ambos sentidos**. ✅
- Verificado en teléfono físico: se escucha la contestadora del número extranjero. **FUNCIONA.**

### 8.2 Llamada NACIONAL (TIGO) — PENDIENTE ⚠️
- Señalización OK: `INVITE sip:3022449235@172.17.179.150` → `100 Trying` → `183 Session Progress` (con PRACK) → `200 OK`. La llamada **conecta** (el celular destino suena).
- Final: `BYE` con `Reason: Q.850;cause=16` = fin normal (colgó el usuario). **No es un error real de señalización.**
- **SDP del UCM hacia TIGO:** `c=IN IP4 192.168.12.170` ← **IP PRIVADA**.
- RTP: UCM → TIGO = **1688 paquetes** (TIGO sí escucha al usuario). TIGO → UCM = **0 paquetes** (el usuario NO escucha nada).
- Verificado en teléfono físico: el celular suena pero no hay audio de regreso. **Confirmado a nivel de troncal (no es Wave).**

**Causa raíz del audio TIGO en una sola vía:**
El UCM anuncia su IP privada `192.168.12.170` en el SDP (por estar TIGO en Local Network Address). TIGO intenta enviar el audio a `192.168.12.170`, que **no puede enrutar** desde su red privada. El enlace /30 de TIGO **solo puede enviar audio a `172.24.249.66`**. TIGO **no hace latching/symmetric RTP** (manda al SDP, no a la IP de origen), por eso devuelve 0 paquetes.

### 8.3 Error "Something is wrong with the remote party's network"
- Apareció en salida TIGO **después de desactivar el SIP ALG**.
- Confirma que el **SIP ALG estaba reescribiendo** la IP privada del SDP (`192.168.12.170` → `172.24.249.66`), lo que hacía funcionar TIGO. Sin ALG, el SDP queda con la IP privada → falla la media.
- **No era códec** (PCMU/PCMA estaban bien configurados).

### 8.4 Sobre la app Wave (WebRTC)
- Mensaje "Local network connection established. Waiting for remote peer connection…" = es del cliente **Wave web (WebRTC)**, capa **distinta** a los troncales.
- El internacional funciona en teléfono físico pero podía fallar en Wave → el problema de Wave es de **media WebRTC/RemoteConnect**, aparte de TIGO. **Pendiente** (ver §10).

---

## 9. EL CONFLICTO CENTRAL (por qué es difícil)

El UCM tiene **una sola "cara externa" global** (External Host + lógica de Local Network Address), pero está detrás de **dos rutas NAT distintas que requieren IPs externas diferentes**:

| Destino | IP externa que el UCM debe anunciar | Vía |
|---|---|---|
| TIGO (`172.17.179.150`) | **`172.24.249.66`** | WAN/LAN1 (enlace /30 privado) |
| Internet / DIDWW | **`185.42.22.239`** | WAN1 (internet) |
| RemoteConnect / Wave | `go.azclegal.com` (`216.238.79.56`) | nube Grandstream |

El **SIP ALG** del ER605 es global (on/off para todo):
- **ALG ON** → arregla TIGO (reescribe IP privada → `172.24.249.66`) … pero puede romper el internacional (handoff del SBC de DIDWW).
- **ALG OFF** → arregla internacional (vía DMZ) … pero rompe TIGO (IP privada en SDP).

→ **Un solo toggle no puede servir a ambos.** De ahí el ping-pong.

---

## 10. Estado actual

| Componente | Estado |
|---|---|
| Internet (WAN1) | ✅ Funciona, DNS público propio |
| DNS / resolución de dominios | ✅ Resuelto (8.8.8.8 en WAN1) |
| Ruta estática a TIGO (`172.17.179.0/24`) | ✅ Corregida (next hop `172.24.249.65`) |
| Ping del ER605 a `172.17.179.150` | ✅ Responde (0% pérdida) |
| Ping del UCM a `172.17.179.150` | ✅ Responde (tras corregir el next hop) |
| Troncal TIGO — señalización / registro | ✅ Conecta, llamadas salen |
| **Troncal TIGO — audio** | ⚠️ **Una sola vía** (no se escucha de regreso) |
| Troncales internacionales (DIDWW) — registro | ✅ Verdes (tras arreglar DNS) |
| **Troncales internacionales — audio** | ✅ **Funciona** (verificado en teléfono físico) |
| App Wave web (WebRTC) | ⚠️ **Pendiente** (media de navegador no levanta) |
| SIP ALG | Desactivado (línea base) |

---

## 11. Pendientes y soluciones recomendadas

### 11.1 Audio de TIGO (una sola vía) — PRINCIPAL

**Opción C (recomendada primero — rápida, sin tocar config):**
Llamar a TIGO y pedir: *"habiliten media latching / Symmetric RTP en su SBC — está enviando el RTP a la dirección del SDP en vez de a la IP de origen."*
Si TIGO hace latching, devolverá el audio a `172.24.249.66` (la IP de origen) → la DMZ lo entrega al UCM → audio resuelto. **Evidencia:** la captura muestra que TIGO devuelve 0 paquetes.

**Opción A (definitiva, bajo control propio — TIGO directo a 2º puerto del UCM):**
- Puerto 2 del UCM (modo Route/Dual): IP estática `172.24.249.66 / 255.255.255.252`, gateway `172.24.249.65`.
- Ruta estática en el UCM: `172.17.179.0/24` vía `172.24.249.65`.
- Así el UCM tiene `172.24.249.66` como interfaz **real** y la anuncia nativa a TIGO. Sin NAT, sin el truco de Local Network, sin depender del ALG.
- **A verificar:** que el UCM6308 permita agregar esa ruta estática (el servidor SIP está detrás del /30). Esto fue lo que complicó el intento inicial por el puerto del UCM.

**Opción B (NO recomendada):** activar SIP ALG en el ER605. Arregla TIGO pero rompe el internacional. Descartada.

### 11.2 App Wave web (WebRTC) — secundario
- El internacional funciona en teléfono físico, así que el problema de Wave es de **media WebRTC / RemoteConnect**, no de los troncales.
- Revisar: configuración de RemoteConnect/UCMRC, puertos de media WebRTC, y que la DMZ/forwarding en WAN1 no haya alterado los puertos que usa Wave.
- Pendiente de atacar una vez cerrado TIGO.

---

## 12. Lecciones / gotchas (para no repetir)

1. **El enlace de TIGO es enrutado, no bridge.** Va en un puerto **WAN** del ER605 (Static IP), no en LAN.
2. **El servidor SIP (`172.17.179.150`) está detrás del gateway (`172.24.249.65`)** → requiere **ruta estática**. No confundir gateway con servidor SIP.
3. **Cuidado con el typo del next hop** (`172.17` vs `172.24`). Un dígito tumbó todo el LAN→TIGO.
4. **La WAN de internet necesita su propio DNS público.** No depender del DNS de TIGO (solo resuelve red interna).
5. **No desactivar Online Detection** del WAN de TIGO esperando que se auto-excluya: su gateway responde, así que siempre sale "Online" y el router le mete tráfico de internet.
6. **Link Backup deja el backup en standby** → no reenvía tráfico LAN por esa WAN. No usarlo para un WAN que debe estar siempre activo para rutas específicas.
7. **El SIP ALG es global y genera conflicto** entre el troncal privado (TIGO) y los de internet. La solución limpia es separar TIGO en su propia interfaz del UCM.
8. **"abnormal/unreachable" en troncales register SIP** puede ser cosmético (servidores *out* que no responden OPTIONS) — confirmar con llamada real, no solo con el dashboard.
9. **Separar la pata del troncal de la pata del teléfono.** Probar siempre con **teléfono físico** antes de culpar al troncal; el Wave web (WebRTC) es otra capa.
10. **Q.850 cause=16 = fin normal de llamada**, no es error. Si "conecta pero no hay audio", el problema es media (RTP/NAT/SDP), no señalización.

---

## 13. Comandos útiles de diagnóstico

**En el ER605 (System Tools → Diagnostics):**
- `Ping 172.17.179.150` con Interface = `WAN/LAN1` → prueba alcance a TIGO con IP de origen `172.24.249.66`.
- `Ping 8.8.8.8` con Interface = `WAN` → prueba internet por WAN1.

**En el UCM (Maintenance → Network Troubleshooting → IP Ping):**
- `ping 172.17.179.150` → alcance LAN→TIGO (a través del router).
- `ping 8.8.8.8` y `ping google.com` → internet + DNS.

**Captura para audio (Maintenance → Network Troubleshooting → Ethernet Capture):**
- Capture Type: Ethernet Capture, Interface: Any, **Filter: vacío** (o `udp`), Storage: Local.
- Start → llamada ~10-15 seg → Stop → Download `.pcap`.
- NO usar el "Analog Signal Trace" (.pcm) — ese es audio analógico FXS/FXO, no sirve para SIP/RTP.

**Análisis del .pcap (Linux):**
```bash
# Mensajes SIP y códigos de respuesta
tcpdump -nnr captura.pcap -A 'udp port 5060' | grep -aoE 'SIP/2.0 [0-9]{3}'
# IPs anunciadas en SDP
tcpdump -nnr captura.pcap -A 'udp port 5060' | grep -aE '^(c=IN|m=audio)'
# Dirección y conteo de RTP con un servidor de media
tcpdump -nnr captura.pcap 'udp and host <IP_MEDIA>' | wc -l
```
