---
tipo: tecnica
fase: explotacion
so: windows
tags: [oscp, tecnica, explotacion, windows]
fecha: 2026-10-05
---

# Cross-Compiling con mingw-w64

> [!abstract] Qué es y cuándo usarla
> Compilar desde **Kali** un exploit/binario escrito para **Windows** (PE) sin necesitar una máquina de ataque Windows. Útil cuando el exploit público está en C con headers de Windows (`winsock2.h`, `windows.h`) y solo tienes Kali. El binario resultante se ejecuta luego desde Kali con **wine**.

## Requisitos previos
```bash
sudo apt install mingw-w64        # cross-compiler para Windows (x86 y x64)
```

## Pasos
```bash
# Compilar a PE de 32 bits (x86)
i686-w64-mingw32-gcc exploit.c -o exploit.exe

# Si falla con undefined reference a funciones de sockets -> enlazar winsock
i686-w64-mingw32-gcc exploit.c -o exploit.exe -lws2_32

# Compilar a PE de 64 bits (x64)
x86_64-w64-mingw32-gcc exploit.c -o exploit.exe -lws2_32

# Ejecutar el PE desde Kali
wine exploit.exe
sudo wine exploit.exe     # si el exploit necesita privilegios (ej. puertos < 1024)
```

| Parte | Qué hace |
|---|---|
| `i686-w64-mingw32-gcc` | Compilador mingw para **x86 (32 bits)** |
| `x86_64-w64-mingw32-gcc` | Compilador mingw para **x64 (64 bits)** |
| `-o <fichero.exe>` | Nombre del PE de salida |
| `-lws2_32` | Enlaza **winsock** (`WSAStartup`, `socket`, `connect`, `send`, `closesocket`...) |
| `wine` | Capa de compatibilidad para correr PEs de Windows en Linux/BSD/macOS |

## Mapear errores del linker → librería a enlazar
Los `undefined reference to '_imp__<Funcion>@N'` **no** son bugs del código: el linker no encuentra la librería. Googlear la **primera** función y añadir su `-l<lib>`:

| Función que aparece | Librería | Flag |
|---|---|---|
| `WSAStartup`, `socket`, `connect`, `send`, `inet_addr`, `htons` | winsock2 | `-lws2_32` |
| `InternetOpenA`, `HttpSendRequest` | wininet | `-lwininet` |

> [!warning] Detección / problemas
> - Usa la **arquitectura correcta** (x86 vs x64) según el target y el shellcode; mezclarlas rompe el exploit.
> - Prefiere el **compilador nativo** del SO cuando puedas; el cross-compile es el plan B cuando solo tienes Kali.
> - Si el exploit abre un **puerto < 1024** o necesita permisos, lánzalo con `sudo wine`.
> - Tras cross-compilar, **prueba en un target controlado** (los offsets/return address siguen dependiendo de la versión exacta).

## Visto en
- Módulo: [[14 - Fixing Exploits]]
- Relacionado: [[SearchSploit]]
- Máquinas: [[]]
