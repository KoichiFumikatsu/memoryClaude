# RDP + SSH desde Linux a Windows

Procedimiento estándar para conectarse desde Fumilinux (Ubuntu 24.04) a una máquina Windows en la misma red local. Usado para banco de pruebas de aplicaciones Windows (ej. cliente AZCKeeper C# .NET 8).

Nota previa: Koichi indica que el setup RDP/SSH a mano no le suele funcionar. Recorrer siempre todos los pasos en orden, no asumir que algún paso ya está hecho.

## Pre-flight checklist

1. **Edición de Windows.** RDP nativo solo existe en Pro / Enterprise / Education. En Home no funciona sin RDP Wrapper.
   - Verificar: `wmic os get caption` o `winver`
   - Si es Home: subir a Pro (clave OEM en BIOS suele activarse sola desde Settings → Activation → Change product key), o usar VNC en lugar de RDP.
2. **Usuario Windows con contraseña real.** RDP rechaza cuentas sin password. PIN de Windows Hello no cuenta.
3. **Ambas máquinas en la misma LAN.** Anotar IP del Windows: `ipconfig` → IPv4 Address.

## Lado Windows

### 1. Activar RDP
- `Settings → System → Remote Desktop → Enable Remote Desktop` (toggle ON)
- Anotar el "PC name"
- `Select users that can remotely access this PC` — agregar la cuenta si no es admin

### 2. Network profile en Private
- `Settings → Network & Internet → [conexión] → Network profile → Private`
- Si está en Public el firewall bloquea 3389 incluso con RDP activado
- **Causa más común de "no conecta" cuando todo lo demás está bien**

### 3. Activar OpenSSH Server (opcional, recomendado)
PowerShell como administrador:
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH SSH Server' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

Default shell de SSH es `cmd.exe`. Cambiar a PowerShell:
```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```

### 4. Power settings (crítico)
`Settings → System → Power & Battery → Screen and sleep`:
- Screen: Never (cuando esté plugged in)
- Sleep: Never (cuando esté plugged in)

Si el portátil duerme, RDP/SSH se cae y hay que despertarlo manualmente.

Tapa cerrada: `Control Panel → Power Options → Choose what closing the lid does → When plugged in: Do nothing`

### 5. IP fija (reserva DHCP o estática)
Reservar la IP del portátil por MAC en el Omada ER707-M2. Si la IP cambia por DHCP, queda inalcanzable sin volver a buscarla.

## Lado Linux (Fumilinux)

### Instalar cliente RDP
```bash
sudo apt install freerdp2-x11 remmina remmina-plugin-rdp
```

### Conectar vía xfreerdp (CLI — usar este para diagnosticar problemas)
```bash
xfreerdp /v:192.168.0.X /u:NombreUsuario /p:'TuPassword' \
  /size:1600x900 /dynamic-resolution +clipboard /sound:sys:pulse \
  /cert:ignore
```

Flags:
- `/v:` IP del portátil
- `/u:` usuario Windows (cuenta Microsoft: `usuario@outlook.com`)
- `/cert:ignore` salta warning de certificado autofirmado
- `/dynamic-resolution` reajusta resolución al redimensionar
- `+clipboard` copy/paste bidireccional
- `/multimon` para multi-monitor si hace falta
- `/sec:nla` o `/sec:tls` si hay problemas de NLA

### Conectar vía Remmina (GUI)
- New connection → Protocol: RDP → Server: IP → Usuario + Password → Color depth 32 → Quality Best

### SSH
```bash
ssh NombreUsuario@192.168.0.X
```

Transferir archivos: `scp -r local/path user@ip:'C:/Users/user/Desktop/dest/'`

## Trampas conocidas

| Síntoma | Causa | Fix |
|---|---|---|
| Connection refused / timeout | Network profile en Public | Cambiar a Private |
| Authentication failure con password correcto | Cuenta Microsoft con 2FA, o sin password local | Crear usuario local con password, o desactivar 2FA |
| Se conecta y se desconecta a los segundos | Windows duerme / pantalla apaga | Power settings → Sleep: Never |
| xfreerdp `ERRCONNECT_LOGON_FAILURE` | NLA mal negociado | `/sec:nla` explícito; alternativa `/sec:tls` |
| Black screen tras login | Driver display o sesión bloqueada | Restart `Remote Desktop Services` en Windows |
| RDP no aparece en Settings | Es Windows Home | Subir a Pro, o usar VNC |
| IP cambia y se pierde | DHCP sin reserva | Reserva en Omada |

## Flujo recomendado para probar AZCKeeper

1. Build en Fumilinux:
   ```bash
   dotnet publish -r win-x64 -c Release --self-contained -p:PublishSingleFile=true
   ```
2. Copiar al portátil:
   ```bash
   scp -r bin/Release/net8.0/win-x64/publish/* user@ip:'C:/Users/user/Desktop/AZCKeeper/'
   ```
3. Conectar vía RDP, ejecutar, probar stealth/KeyBlocker/handshake
4. Logs del cliente en `%AppData%\AZCKeeper\Logs\YYYY-MM-DD.log` — leíbles vía SSH sin abrir GUI
