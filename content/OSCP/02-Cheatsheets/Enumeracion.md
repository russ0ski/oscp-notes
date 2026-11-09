---
tipo: cheatsheet
tags: [oscp, cheatsheet]
---

# Cheatsheet: Enumeracion

> [!info] Comandos añadidos a medida que avanzo en los módulos.

## Whois / OSINT
```bash
whois megacorpone.com -h 192.168.50.251     # forward: dominio -> propietario
whois 38.100.193.70 -h 192.168.50.251       # reverse: IP -> ISP/CIDR
```

## Google Dorks
```text
site:dominio.com filetype:txt               # ficheros por tipo (encontró robots.txt)
site:dominio.com -filetype:html             # páginas no-HTML / index de directorios
intitle:"index of" "parent directory"       # directory listing
site:dominio.com ext:env | ext:conf | ext:ini
```
Ver: GHDB (Exploit-DB), DorkSearch.
```text
site:example.com filetype:pdf               # docs públicos para leer metadatos (client-side recon)
"<app> <versión>" exploit                    # buscar exploits públicos
firefox --search "Microsoft Edge site:exploit-db.com"   # limitar a exploit-db.com (operadores: inurl, intext, intitle)
```

## Metadatos de documentos (exiftool)
```bash
exiftool -a -u brochure.pdf                 # -a: tags duplicados  -u: tags desconocidos
# Interesan: Author/Creator (empleado), Producer/Creator Tool (SO+software), Create/Modify Date
```

## Repos públicos (GitHub, etc.)
```bash
# En GitHub UI:  owner:megacorpone path:users   -> xampp.users (hash APR1)
gitleaks-linux-amd64 -v -r=https://github.com/<user>/<repo>   # secretos por regex/entropía
```

## DNS
```bash
host www.megacorpone.com                    # A por defecto
host -t mx megacorpone.com                  # MX
host -t txt megacorpone.com                 # TXT
for ip in $(cat list.txt); do host $ip.megacorpone.com; done          # brute forward
for ip in $(seq 64 79); do host 167.114.21.$ip; done | grep -Ev "not found|timed out"  # PTR
dnsrecon -d megacorpone.com -t std
dnsrecon -d megacorpone.com -D ~/list.txt -t brt
dnsenum megacorpone.com
gobuster dns -d megacorpone.com -w wordlist.txt -t 10
```
```powershell
nslookup mail.megacorptwo.com
nslookup -type=TXT info.megacorptwo.com 192.168.50.151
```

## Port scanning (Netcat)
```bash
nc -nvv -w 1 -z 192.168.50.152 3388-3390    # TCP scan (-z zero-I/O, -w timeout)
nc -nv -u -z -w 1 192.168.50.149 120-123    # UDP scan (-u)
```

## Port scanning (Nmap)
```bash
nmap 192.168.50.149                          # top 1000 TCP
nmap -p 1-65535 192.168.50.149               # todos los TCP
sudo nmap -sS 192.168.50.149                 # SYN / stealth
nmap -sT 192.168.50.149                      # TCP connect (sin root)
sudo nmap -sU 192.168.50.149                 # UDP
sudo nmap -sU -sS 192.168.50.149             # UDP + SYN
nmap -sn 192.168.50.1-253 -oG ping-sweep.txt # ping sweep
grep Up ping-sweep.txt | cut -d " " -f 2     # hosts vivos
nmap -p 80 192.168.50.1-253 -oG web-sweep.txt
nmap -sT -A --top-ports=20 192.168.50.1-253 -oG top-port-sweep.txt
sudo nmap -O 192.168.50.14 --osscan-guess    # OS fingerprint
nmap -sV 192.168.50.14                        # versiones
nmap -sT -A 192.168.50.14                     # agresivo
nmap --script http-headers 192.168.50.6       # NSE
nmap --script-help http-headers
```
```powershell
Test-NetConnection -Port 445 192.168.50.151
1..1024 | % {echo ((New-Object Net.Sockets.TcpClient).Connect("192.168.50.151", $_)) "TCP port $_ is open"} 2>$null
```

## SMB / NetBIOS
```bash
nmap -v -p 139,445 -oG smb.txt 192.168.50.1-254
sudo nbtscan -r 192.168.50.0/24
nmap -v -p 139,445 --script smb-os-discovery 192.168.50.152
ls -1 /usr/share/nmap/scripts/smb*
```
```powershell
net view \\dc01 /all
```

## SMTP
```bash
nc -nv 192.168.50.8 25       # luego: VRFY root  (252 existe / 550 no existe)
python3 smtp.py root 192.168.50.8
```
```powershell
dism /online /Enable-Feature /FeatureName:TelnetClient
telnet 192.168.50.8 25       # VRFY <user>
```

## SNMP
```bash
sudo nmap -sU --open -p 161 192.168.50.1-254 -oG open-snmp.txt
onesixtyone -c community -i ips                                  # brute community strings
snmpwalk -c public -v1 -t 10 192.168.50.151                     # todo el MIB
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.4.1.77.1.2.25     # usuarios
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.2.1.25.4.2.1.2    # procesos
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.2.1.25.6.3.1.2    # software
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.2.1.6.13.1.3      # puertos TCP
```

## Vulnerability Scanning

### Nessus (instalación Kali)
```bash
cd ~/Downloads
echo "<sha256> Nessus-10.5.0-debian10_amd64.deb" > sha256sum_nessus  # hash + nombre del fichero
sha256sum -c sha256sum_nessus                        # verificar integridad -> OK
sudo apt install ./Nessus-10.5.0-debian10_amd64.deb  # instalar .deb
sudo systemctl start nessusd.service                 # arrancar (UI en https://127.0.0.1:8834)
sudo apt purge nessus && sudo rm -rf /opt/nessus     # purga total (reset del límite de 5 IPs)
```
Plantillas: **Basic Network Scan** (recomendada), **Credentialed Patch Audit** (autenticado, checks locales), **Advanced Dynamic Scan** (filtro dinámico de plugins por CVE/family). Auth: Linux→SSH, Windows→SMB/WMI.

### Nmap NSE (vuln)
```bash
cat /usr/share/nmap/scripts/script.db | grep "\"vuln\""            # scripts de la categoría vuln
sudo nmap -sV -p 443 --script "vuln" 192.168.50.13                 # todos vuln (incl. vulners: CVE+CVSS+*EXPLOIT*)
sudo cp ~/Downloads/http-vuln-cve-2021-41773.nse /usr/share/nmap/scripts/http-vuln-cve2021-41773.nse
sudo nmap --script-updatedb                                        # registrar script nuevo
sudo nmap -sV -p 443 --script "http-vuln-cve2021-41773" 192.168.50.124   # check de un CVE concreto
grep Exploits /usr/share/nmap/scripts/*.nse        # scripts NSE que "explotan" algo
nmap --script-help=clamav-exec.nse                 # qué hace un script (categorías, CVE, descripción)
```

## Búsqueda de exploits (searchsploit / Exploit-DB)  — ver [[SearchSploit]]
```bash
sudo apt update && sudo apt install exploitdb      # actualizar copia local ANTES del engagement
ls -1 /usr/share/exploitdb/                        # exploits/ shellcodes/ + CSV de referencia
searchsploit remote smb microsoft windows          # buscar (AND, case-insensitive, rango de versiones)
searchsploit -t qdpm --exclude="(PoC)|/dos/"       # solo título, excluir ruido
searchsploit -m 50944                              # copiar (mirror) el exploit al CWD por EDB-ID
searchsploit -p 50944                              # ruta completa (+ portapapeles)   | -w URL  | -j JSON
```
Online: **Exploit-DB** (campos D/A/V, verificado), **Packet Storm** (curados); **GitHub** (no curado, cautela). EDB-ID ≠ CVE. Leer SIEMPRE el código antes de ejecutar.

