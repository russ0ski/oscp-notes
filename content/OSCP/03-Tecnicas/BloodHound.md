---
tipo: tecnica
fase: enumeracion
so: windows
tags: [oscp, tecnica, ad, enumeracion, windows]
---

# BloodHound

> [!abstract] Qué es y cuándo usarla
> **SharpHound** recolecta datos del dominio y **BloodHound** (sobre Neo4j) los grafica para revelar **rutas de ataque** (shortest path a Domain Admins). Úsalo tras tener un punto de apoyo en el dominio para visualizar relaciones que la enumeración manual pasa por alto.

## Requisitos previos
- Usuario de dominio y un host donde ejecutar SharpHound (Windows).
- En Kali: `neo4j` y `bloodhound` instalados (APT instala Neo4j como dependencia).
- Consciencia de que **genera mucho ruido** de red.

## Pasos
```powershell
# 1) Recolección (en el host de dominio)
powershell -ep bypass
Import-Module .\Sharphound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory . -OutputPrefix "audit"
#   -> .zip con JSON (el .bin es caché, se puede borrar). -Loop para snapshots periódicos.
```
```bash
# 2) Análisis (en Kali)
sudo neo4j start                 # http://localhost:7474 -> login neo4j/neo4j -> cambiar password
bloodhound                       # cargar el .zip con "Upload Data"
```
En la GUI:
- **Analysis** → *Find all Domain Admins*, *Find Shortest Paths to Domain Admins*.
- Marcar lo que controlas: **Mark User/Computer as Owned** (icono calavera) → *Shortest Paths to Domain Admins from Owned Principals*.
- Aristas = relación (`AdminTo`, `MemberOf`, `HasSession`, `GenericAll`...). Botón derecho → `? Help` → pestañas **Abuse** (cómo explotar) y **Opsec** (cómo te detectan).

> [!tip] Marca como *owned* **todo** lo que controles: puede aparecer una ruta corta que dependa de poseer ese objeto. Sin owned principals, la query de "from Owned" devuelve "NO DATA".

> [!warning] Detección / problemas
> - `CollectionMethod All` + sesiones = **pico de tráfico** fácilmente detectable.
> - SharpHound usa NetSessionEnum/Remote Registry → en Windows moderno algunas sesiones no se recogen.
> - Usa la versión de SharpHound **compatible** con tu versión de BloodHound.

## Visto en
- Módulo: [[22 - Active Directory Introduction and Enumeration]]
- Máquinas: [[]]
