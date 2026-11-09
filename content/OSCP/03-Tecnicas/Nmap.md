---
tipo: tecnica
tags: [oscp, enumeracion, linux]
fecha: 2026-09-26
---

# Nmap

> [!abstract]
> Escáner de puertos de facto. Descubre puertos abiertos, servicios, versiones y SO. Muchas opciones requieren `sudo` (raw sockets).

## Tipos de scan
```bash
nmap 192.168.50.149                 # top 1000 puertos TCP (por defecto)
nmap -p 1-65535 192.168.50.149      # todos los puertos TCP (-p- equivalente)
sudo nmap -sS 192.168.50.149        # SYN / "stealth" (por defecto con root)
nmap -sT 192.168.50.149             # TCP connect (sin privilegios, más lento)
sudo nmap -sU 192.168.50.149        # UDP
sudo nmap -sU -sS 192.168.50.149    # UDP + SYN combinados
```
| Flag | Qué hace |
|---|---|
| `-sS` | SYN scan; no completa handshake, rápido, no llega a app layer |
| `-sT` | Connect scan; API sockets, sin root, más lento |
| `-sU` | UDP scan (lento, poco fiable) |
| `-p` / `-p-` | Puertos concretos / todos (1-65535) |
| `-Pn` | Sin host discovery, trata todo como vivo (hosts que bloquean ICMP) |

## Descubrimiento y versiones
```bash
nmap -sn 192.168.50.1-253 -oG ping-sweep.txt      # ping sweep (SYN 443 + ACK 80 + ICMP)
grep Up ping-sweep.txt | cut -d " " -f 2           # extraer hosts vivos
nmap -p 80 192.168.50.1-253 -oG web-sweep.txt      # sweep por puerto
nmap -sT -A --top-ports=20 192.168.50.1-253 -oG top-port-sweep.txt
sudo nmap -O 192.168.50.14 --osscan-guess          # OS fingerprint
nmap -sV 192.168.50.14                              # solo versiones de servicio
nmap -sT -A 192.168.50.14                           # versión + scripts + OS + traceroute
```
| Flag | Qué hace |
|---|---|
| `-sn` | Solo host discovery (no port scan) |
| `-O` / `--osscan-guess` | Fingerprint SO / forzar mejor conjetura |
| `-sV` | Detección de versiones |
| `-A` | Agresivo: versión + NSE + OS + traceroute (mucho tráfico) |
| `-oG <f>` | Salida greppable |
| `--top-ports=N` | N puertos más comunes |

## NSE (Nmap Scripting Engine)
```bash
nmap --script http-headers 192.168.50.6      # ejecutar un script
nmap --script-help http-headers              # info del script
ls -1 /usr/share/nmap/scripts/smb*           # scripts disponibles
```

## NSE como vulnerability scanner
```bash
cd /usr/share/nmap/scripts/
cat script.db | grep "\"vuln\""                       # scripts de la categoría vuln
sudo nmap -sV -p 443 --script "vuln" 192.168.50.13    # todos los scripts vuln (incl. vulners)
```
`vulners` (categorías `safe,vuln,external`) mapea la versión detectada → CVEs + CVSS + enlaces; PoCs marcados con `*EXPLOIT*`. **Requiere `-sV` con detección de versión exitosa.**

Script NSE a medida para un CVE concreto:
```bash
sudo cp ~/Downloads/http-vuln-cve-2021-41773.nse /usr/share/nmap/scripts/http-vuln-cve2021-41773.nse
sudo nmap --script-updatedb                                                    # registrar en script.db
sudo nmap -sV -p 443 --script "http-vuln-cve2021-41773" 192.168.50.124         # buscar VULNERABLE
```
| Categoría | Implicación |
|---|---|
| `safe` | Sin impacto en estabilidad |
| `intrusive` | Puede tumbar el servicio/sistema |
| `vuln` | Detección de vulnerabilidades |

> [!warning]
> Un `.nse` de terceros puede contener código malicioso. Verifica que hace lo que dice **y** que es seguro antes de ejecutarlo.

> [!tip]
> El scan por defecto usa `/usr/share/nmap/nmap-services` (frecuencia de cada puerto en internet). Un scan `-p-` de una /24 puede generar ~1 GB — balancea cobertura vs ruido/tiempo.

> [!warning]
> OS fingerprinting y banners son **aproximados**: firewalls/proxies reescriben headers y los admins pueden falsear banners.

## Relacionado
- Módulo: [[06 - Information Gathering]]
- Cheatsheet: [[Enumeracion]]
