---
tipo: tecnica
tags: [oscp, enumeracion, windows]
fecha: 2026-09-26
---

# SMB Enumeration

> [!abstract]
> SMB (**TCP/445**) y NetBIOS (**TCP/139**) son protocolos distintos pero se enumeran juntos (NBT por compatibilidad). Largo historial de fallos: null sessions, vulns, shares expuestos.

## Descubrir el servicio
```bash
nmap -v -p 139,445 -oG smb.txt 192.168.50.1-254   # buscar hosts con SMB/NetBIOS
sudo nbtscan -r 192.168.50.0/24                    # nombres NetBIOS (puerto origen 137)
```
| Flag | Qué hace |
|---|---|
| `-p 139,445` | Puertos NetBIOS + SMB |
| `-r` (nbtscan) | Usa UDP/137 como puerto origen |

> [!tip]
> Los **nombres NetBIOS** suelen describir el rol del host (SAMBA, SAMBAWEB...): pistas para priorizar objetivos.

## NSE scripts
```bash
ls -1 /usr/share/nmap/scripts/smb*
nmap -v -p 139,445 --script smb-os-discovery 192.168.50.152
```
Scripts útiles: `smb-os-discovery`, `smb-enum-shares`, `smb-enum-users`, `smb-enum-groups`, `smb2-security-mode`, `smb-vuln-*`.

> [!warning]
> Muchos scripts (`smb-os-discovery` incluido) solo funcionan si **SMBv1** está habilitado (raro en Windows moderno). Y la detección de SO por SMB puede fallar (reportó Win10 en un Win11).

## Windows / LOLBAS: net view
```powershell
net view \\dc01 /all
```
`/all` lista también los shares **administrativos** (terminados en `$`: `ADMIN$`, `C$`, `IPC$`).

## Relacionado
- Módulo: [[06 - Information Gathering]]
- Cheatsheet: [[Enumeracion]]
