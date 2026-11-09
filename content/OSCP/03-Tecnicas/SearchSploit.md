---
tipo: tecnica
fase: explotacion
so: multi
tags: [oscp, tecnica, explotacion]
fecha: 2026-10-03
---

# SearchSploit

> [!abstract] Qué es y cuándo usarla
> CLI para buscar en la **copia local de Exploit-DB** que trae Kali (paquete `exploitdb`). Sirve **offline** y acelera el cruce "servicio+versión → exploit". Úsala tras enumerar la app y su **versión exacta**.

## Requisitos previos
- Paquete `exploitdb` instalado y actualizado:
```bash
sudo apt update && sudo apt install exploitdb      # actualizar ANTES de cada engagement
```
- Archivo local en `/usr/share/exploitdb/` (`exploits/`, `shellcodes/` + CSV de referencia).

## Pasos
```bash
searchsploit remote smb microsoft windows          # buscar (AND de términos, case-insensitive, rango de versiones)
searchsploit -t qdpm                                # buscar solo en el título (menos falsos positivos)
searchsploit qdPM 9.1 --exclude="(PoC)|/dos/"       # excluir ruido
searchsploit -m 50944                               # copiar (mirror) el exploit al CWD por EDB-ID
searchsploit -m php/webapps/50944.py                # ...o por ruta
searchsploit -p 50944                               # ruta completa + al portapapeles
searchsploit -w qdpm                                # mostrar URL de exploit-db.com
searchsploit -j qdpm | json_pp                      # salida JSON
```

| Flag | Qué hace |
|---|---|
| `-m <ruta/EDB-ID>` | Copia el exploit al CWD (editar la copia, no el original) |
| `-t` | Busca solo en el **título** (evita match por la ruta; útil con versiones) |
| `-e` | Match **exacto** y ordenado del título |
| `-s` | **Strict**: versión exacta (sin rango difuso) |
| `-c` | Búsqueda **case-sensitive** |
| `-p <EDB-ID>` | Ruta completa (+ copia al portapapeles) |
| `--exclude="t1\|t2"` | Excluir términos |
| `-j` / `-w` | Salida JSON / URL de Exploit-DB |

> [!warning] Detección / problemas
> - Al reinstalar/actualizar `exploitdb` se **sobrescriben** los exploits locales → trabaja siempre sobre la **copia** (`-m`), nunca sobre `/usr/share/exploitdb/`.
> - Buscar por número de versión da **falsos positivos** por la ruta → combina `-t` (solo título) y `-s` (versión estricta).
> - Lee **siempre** el código antes de ejecutarlo (args que pide, qué sube/ejecuta).

## Visto en
- Módulo: [[13 - Locating Public Exploits]]
- Relacionado: [[Nmap]] (NSE `exploit`/`vuln`), [[Google Dorks]]
- Máquinas: [[]]
