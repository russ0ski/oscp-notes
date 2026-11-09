---
tipo: tecnica
fase: enumeracion
so: windows
tags: [oscp, tecnica, ad, enumeracion, windows]
---

# Enumeración AD con LDAP y .NET

> [!abstract] Qué es y cuándo usarla
> Enumerar Active Directory con **PowerShell + clases .NET + ADSI** (LDAP) sin depender de RSAT/`Get-ADUser` (que solo están en DCs). Útil cuando no hay PowerView disponible o quieres entender cómo funcionan por dentro las herramientas; solo requiere privilegios básicos.

## Requisitos previos
- Sesión de un usuario de dominio en un host del dominio.
- `powershell -ep bypass` para cargar el script.

## Pasos
```powershell
# 1) Construir el path LDAP dinámicamente (apuntando al PDC)
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
$DN  = ([adsi]'').distinguishedName        # p.ej. DC=corp,DC=com
$LDAP = "LDAP://$PDC/$DN"                   # LDAP://DC1.corp.com/DC=corp,DC=com

# 2) Buscar con DirectoryEntry + DirectorySearcher
$direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)
$dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
$dirsearcher.filter="samAccountType=805306368"   # solo usuarios (0x30000000)
$result = $dirsearcher.FindAll()

# 3) Imprimir atributos (bucle anidado)
Foreach($obj in $result){ Foreach($prop in $obj.Properties){ $prop }; Write-Host "---" }
```
Como función reutilizable:
```powershell
function LDAPSearch {
    param ([string]$LDAPQuery)
    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DistinguishedName = ([adsi]'').distinguishedName
    $DirectoryEntry = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$PDC/$DistinguishedName")
    $DirectorySearcher = New-Object System.DirectoryServices.DirectorySearcher($DirectoryEntry, $LDAPQuery)
    return $DirectorySearcher.FindAll()
}
```
```powershell
Import-Module .\function.ps1
LDAPSearch -LDAPQuery "(samAccountType=805306368)"               # usuarios
LDAPSearch -LDAPQuery "(objectclass=group)"                      # grupos
$g = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=Sales Department))"
$g.properties.member                                             # revela nested groups
```

> [!tip] El **DN se lee de derecha a izquierda** (`CN=Stephanie,CN=Users,DC=corp,DC=com`). Filtra por cualquier atributo (`name=jeffadmin`) y encadena con `(&(...)(...))` para AND.

> [!warning] Detección / problemas
> - `Get-ADUser` (RSAT) solo está en DCs y requiere admin para instalarlo → por eso este método.
> - Sin filtro, `FindAll()` devuelve **todo** el dominio (mucha salida).

## Visto en
- Módulo: [[22 - Active Directory Introduction and Enumeration]]
- Máquinas: [[]]
