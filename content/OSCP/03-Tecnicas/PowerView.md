---
tipo: tecnica
fase: enumeracion
so: windows
tags: [oscp, tecnica, ad, enumeracion, windows]
---

# PowerView

> [!abstract] Qué es y cuándo usarla
> Script PowerShell para **enumerar Active Directory** vía LDAP (igual que un script .NET/ADSI casero, pero con muchas funciones listas). Úsalo desde un host de dominio con un usuario (aunque sea de bajo privilegio) para mapear users, groups, computers, sesiones, SPNs, ACLs y shares.

## Requisitos previos
- Sesión en un host del dominio (idealmente por **RDP** para evitar el Kerberos double-hop).
- Cargarlo en memoria: `powershell -ep bypass` + `Import-Module .\PowerView.ps1`.

## Pasos
```powershell
Import-Module .\PowerView.ps1

# Dominio / usuarios / grupos
Get-NetDomain
Get-NetUser | select cn,pwdlastset,lastlogon       # atributos (dormant / password viejo)
Get-NetGroup | select cn
Get-NetGroup "Sales Department" | select member     # ve nested groups (net.exe no)

# Equipos y acceso
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
Find-LocalAdminAccess                               # ¿dónde soy admin local? (spray)
Get-NetSession -ComputerName <host> -Verbose        # sesiones (suele fallar en Win moderno)

# Service accounts (SPN) -> base de Kerberoasting
Get-NetUser -SPN | select samaccountname,serviceprincipalname

# Permisos / ACLs
Get-ObjectAcl -Identity "<obj>" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
Convert-SidToName <SID>

# Shares
Find-DomainShare -CheckShareAccess                  # sin flag = todos los shares
```

> [!tip] Filtra siempre con `select` (y `?{...}`) para no ahogarte en atributos. `Convert-SidToName` traduce los SID de las ACLs a nombres legibles.

> [!warning] Detección / problemas
> - Genera consultas LDAP visibles; SharpHound/PowerView son ruidosos.
> - `Get-NetSession` da "Access denied" en Win10 1709+/Server 2019+ → usa [[BloodHound]]/PsLoggedOn.
> - Requiere política de ejecución relajada (`-ep bypass`).

## Visto en
- Módulo: [[22 - Active Directory Introduction and Enumeration]]
- Máquinas: [[]]
