---
tipo: tecnica
tags: [oscp, enumeracion, linux, windows]
fecha: 2026-09-26
---

# DNS Enumeration

> [!abstract]
> Consultar DNS para descubrir hostnames, IPs, servidores de correo y name servers. Objetivo muy rico en info. Activo (interactúa con el DNS).

## Tipos de registro
| Registro | Contenido |
|---|---|
| NS | Servidores autoritativos del dominio |
| A / AAAA | IPv4 / IPv6 de un hostname |
| MX | Servidores de correo (con prioridad; menor = primero) |
| PTR | Reverse lookup (IP → nombre) |
| CNAME | Alias de otro host |
| TXT | Datos arbitrarios (verificación de ownership, SPF...) |

## Consultas puntuales (`host`)
```bash
host www.megacorpone.com                # A por defecto
host -t mx megacorpone.com              # registros MX
host -t txt megacorpone.com             # registros TXT
```
| Flag | Qué hace |
|---|---|
| `-t <tipo>` | Fija el tipo de registro (mx, txt, ns...) |

Host válido → resuelve a IP. Host inexistente → `NXDOMAIN` (así distinguimos válidos de inválidos).

## Brute-force con Bash
```bash
# Forward: probar hostnames de un wordlist
for ip in $(cat list.txt); do host $ip.megacorpone.com; done

# Reverse (PTR): barrer un rango contiguo, filtrar ruido
for ip in $(seq 64 79); do host 167.114.21.$ip; done | grep -Ev "not found|timed out"
```
> [!tip]
> Los subdominios suelen agruparse en un mismo rango (ej. `167.114.21.X`). Si encuentras varios A ahí, barre PTR de todo el bloque. Wordlists: `sudo apt install seclists` → `/usr/share/seclists`.

## Herramientas automáticas
```bash
dnsrecon -d megacorpone.com -t std                 # scan estándar (NS, MX, A, TXT, SOA...)
dnsrecon -d megacorpone.com -D ~/list.txt -t brt   # brute de subdominios
dnsenum megacorpone.com                             # automatiza + intenta zone transfer
gobuster dns -d megacorpone.com -w wordlist.txt -t 10
```
| Flag | Qué hace |
|---|---|
| `-d` | Dominio objetivo |
| `-D` | Wordlist de subdominios (dnsrecon) |
| `-t std/brt/axfr` | Tipo de enumeración (dnsrecon) |
| `-w` / `-t` | Wordlist / threads (gobuster) |

> [!warning]
> `gobuster` >3.6 usa `--do` en vez de `-d` para el dominio. Ante la duda: `gobuster dns --help`.

## Windows / LOLBAS: nslookup
```powershell
nslookup mail.megacorptwo.com
nslookup -type=TXT info.megacorptwo.com 192.168.50.151   # tipo + servidor DNS concreto
```

> [!question] Zone transfer (AXFR)
> Un DNS mal configurado puede volcar toda la zona. `dnsenum` y `dnsrecon -t axfr` lo intentan; también `host -l dominio ns-server`.

## Relacionado
- Módulo: [[06 - Information Gathering]]
- Cheatsheet: [[Enumeracion]]
