---
tipo: tecnica
tags: [oscp, enumeracion, linux, windows]
fecha: 2026-09-26
---

# SMTP User Enumeration

> [!abstract]
> SMTP (**TCP/25**) soporta `VRFY` (verifica una dirección) y `EXPN` (miembros de una lista). En servidores mal configurados sirven para **enumerar usuarios válidos** por la diferencia de respuestas.

## Manual con netcat
```bash
nc -nv 192.168.50.8 25
VRFY root          # 252 2.0.0 root                        -> aceptado
VRFY idontexist    # 550 5.1.1 ... User unknown            -> no existe
```
| Código | Significado |
|---|---|
| `252` | Aceptado / intentará entregar (usuario probable) |
| `550` | Mailbox no disponible (usuario desconocido) |

## Script Python (VRFY)
```python
#!/usr/bin/python
import socket, sys
if len(sys.argv) != 3:
    print("Usage: vrfy.py <username> <target_ip>"); sys.exit(0)
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((sys.argv[2], 25))
print(s.recv(1024))                       # banner
s.send(b'VRFY ' + sys.argv[1].encode() + b'\r\n')
print(s.recv(1024))                       # respuesta
s.close()
```
```bash
python3 smtp.py root 192.168.50.8
```

## Windows / LOLBAS
```powershell
Test-NetConnection -Port 25 192.168.50.8              # solo comprueba si el puerto abre
dism /online /Enable-Feature /FeatureName:TelnetClient # instalar Telnet (requiere admin)
```
```text
telnet 192.168.50.8 25
VRFY root
```
> [!warning]
> `Test-NetConnection` no permite interactuar con SMTP (solo TCP test). Para `VRFY` necesitas Telnet; si no eres admin, transfiere `telnet.exe` desde otra máquina.

## Relacionado
- Módulo: [[06 - Information Gathering]]
- Cheatsheet: [[Enumeracion]]
