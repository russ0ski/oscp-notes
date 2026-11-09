---
tipo: tecnica
fase: pivoting
so: linux
tags: [oscp, tecnica, pivoting, linux, windows]
---

# SSH Port Forwarding

> [!abstract] Qué es y cuándo usarla
> Encapsular tráfico dentro de una conexión SSH para alcanzar servicios internos no rutables desde Kali. Cuatro variantes según **dónde se bindea el puerto de escucha** y **quién reenvía**. Elige según lo que permita el firewall.

## Requisitos previos
- Credenciales SSH (o shell) en un host pivote con visibilidad a la red interna.
- Para las variantes **remote**: servidor SSH corriendo en Kali (`sudo systemctl start ssh`, `PasswordAuthentication yes`).
- Para **dynamic**/**remote dynamic**: herramientas que hablen SOCKS o [[Proxychains]]. Remote dynamic necesita **OpenSSH cliente ≥7.6**.

## Pasos
```bash
# LOCAL (-L): escucha en el CLIENTE, reenvía el server. Un socket.
#   -L [LOCAL_IP:]LOCAL_PORT:DEST_IP:DEST_PORT
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 user@PIVOT

# DYNAMIC (-D): SOCKS en el CLIENTE. Cualquier host/puerto.
ssh -N -D 0.0.0.0:9999 user@PIVOT
# proxychains4.conf -> socks5 <IP_PIVOT> 9999

# REMOTE (-R): escucha en el SERVIDOR (Kali), reenvía el cliente. Para firewall inbound.
ssh -N -R 127.0.0.1:2345:10.4.50.215:5432 kali@KALI

# REMOTE DYNAMIC (-R <port>): SOCKS en el SERVIDOR (Kali). Mismo -R, un solo puerto.
ssh -N -R 9998 kali@KALI
# proxychains4.conf -> socks5 127.0.0.1 9998
```
Regla mnemotécnica: **local/dynamic → puerto en el CLIENTE** (estás dentro, abres puerto en el pivote); **remote/remote-dynamic → puerto en el SERVIDOR = Kali** (el pivote solo puede salir). `-N` = sin shell.

### Windows
```powershell
ssh -N -R 9998 kali@KALI          # ssh.exe (OpenSSH nativo ≥1803, ≥7.6 permite remote dynamic)
plink.exe -ssh -l kali -pw <PASS> -R 127.0.0.1:9833:127.0.0.1:3389 KALI   # Plink (no remote dynamic)
```

> [!warning] Detección / problemas
> - Usuario no privilegiado no puede escuchar en puertos **<1024**.
> - El forward ocupa la shell → saca otra revshell para enumerar / `ss -ntplu`.
> - Verifica con `ss -ntplu` dónde quedó el puerto (cliente vs loopback de Kali).
> - Shell sin TTY no acepta el prompt de host key → `python3 -c 'import pty; pty.spawn("/bin/bash")'`.

## Visto en
- Módulo: [[19 - Port Redirection and SSH Tunneling]]
- Máquinas: [[]]
