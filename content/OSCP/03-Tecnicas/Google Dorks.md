---
tipo: tecnica
tags: [oscp, enumeracion]
fecha: 2026-09-26
---

# Google Dorks

> [!abstract]
> Operadores de búsqueda para encontrar info sensible, ficheros y páginas mal configuradas de un objetivo. Pasivo. Proceso iterativo: empezar amplio y estrechar.

## Operadores clave
| Operador | Qué hace |
|---|---|
| `site:` | Limita a un dominio/subdominios |
| `filetype:` / `ext:` | Solo un tipo de fichero (`txt`, `php`, `xml`, `py`...) |
| `-` | Excluye términos (ej. `-filetype:html`) |
| `intitle:"..."` | Texto en el título de la página |
| `inurl:"..."` | Texto en la URL (`admin`, `login`, `upload`...) |
| `cache:` | Versión cacheada de una página |
| `related:` | Sitios similares (proveedores, terceros) |

## Ejemplos
```text
site:megacorpone.com                              # presencia web general
site:megacorpone.com filetype:txt                 # encontró robots.txt -> /nanites.php
site:megacorpone.com -filetype:html               # páginas no-HTML, índices de directorio
intitle:"index of" "parent directory"             # directory listing (sin index)
site:megacorpone.com ext:env | ext:conf | ext:ini # ficheros de config
site:megacorpone.com "username" "password"        # posibles credenciales filtradas
```

> [!tip]
> `ext:php`, `ext:xml`, `ext:py`... revelan qué lenguajes usa el sitio. Los *directory listings* (`index of`) suelen exponer ficheros sensibles.

## Recursos
- **GHDB** (Google Hacking Database, Exploit-DB): dorks prefabricados.
- **DorkSearch**: builder y subset de queries.

## Relacionado
- Módulo: [[06 - Information Gathering]]
- Cheatsheet: [[Enumeracion]]
