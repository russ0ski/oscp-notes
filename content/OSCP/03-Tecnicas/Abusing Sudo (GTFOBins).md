---
tipo: tecnica
fase: privesc
so: linux
tags: [oscp, tecnica, privesc, linux]
fecha: 2026-10-08
---

# Abusing Sudo (GTFOBins)

> [!abstract] Qué es y cuándo usarla
> Si `sudo -l` muestra que el user actual puede ejecutar ciertos binarios como root (sin password o con ella), y alguno de esos binarios aparece en **GTFOBins** como explotable vía sudo, se puede usar esa funcionalidad (lectura/escritura de ficheros arbitraria, o escape a shell) para obtener una shell de root.

## Requisitos previos
- Conocer qué comandos permite sudo al user actual: `sudo -l`.
- Consultar `gtfobins.github.io/gtfobins/<binario>/#sudo` para cada binario permitido.
- Si el primer intento falla de forma inesperada, revisar si hay **AppArmor** (`aa-status`, requiere root o lectura del perfil) o **auditd** bloqueando el binario, mirando `/var/log/syslog`.

## Pasos
```bash
sudo -l                                   # lista de comandos permitidos (y si piden password)
```
Ejemplo con `apt-get` (GTFOBins):
```bash
sudo apt-get changelog apt                 # abre el pager 'less'
!/bin/sh                                     # dentro de less, escapa a una shell con privilegios de sudo
```
Si un vector esperado falla (ej. `tcpdump`), comprobar bloqueo por AppArmor:
```bash
cat /var/log/syslog | grep <binario>          # buscar "apparmor=DENIED"
su - root ; aa-status                          # (si se puede) ver perfiles activos
```

> [!warning] Detección / problemas
> - No asumir que el primer comando de GTFOBins funcionará: **AppArmor** puede bloquear binarios concretos (ej. `tcpdump`) aunque `sudo -l` lo permita — probar el siguiente comando permitido.
> - El intento queda registrado en `/var/log/syslog`/auditd; en un entorno real esto es ruido detectable.
> - Comprobar siempre la sección `#sudo` específica de GTFOBins, no la de SUID (los payloads cambian).

## Visto en
- Módulo: [[18 - Linux Privilege Escalation]]
- Relacionado: [[SUID y Capabilities (Linux)]]
- Máquinas: [[]]
