---
tipo: tecnica
fase: ad
so: windows
tags: [oscp, tecnica, ad, password-attacks]
fecha: 2026-10-09
---

# CrackMapExec - Spraying de Credenciales

> [!abstract] Qué es y cuándo usarla
> CrackMapExec (CME) permite probar una o varias combinaciones usuario/contraseña contra **muchos hosts a la vez** vía SMB, distinguiendo credenciales válidas de inválidas por el código de error devuelto, y — una vez autenticado — enumerar permisos sobre los recursos compartidos de cada host de golpe. Es el paso natural tras recolectar varias credenciales/usuarios dispersos (de distintas fases de un engagement) y no saber contra qué host o cuenta encajan.

## Requisitos previos
- Conectividad SMB (445/tcp) hacia los hosts objetivo.
- Una o más credenciales (usuario+password, o varias listas) a probar; opcionalmente el dominio (`-d`).

## Pasos
```bash
# Spraying de una credencial contra varios hosts
crackmapexec smb <IP1> <IP2> <IP3> -u <user> -d <dominio> -p '<password>' --continue-on-success
# Spraying de listas completas (usuarios x passwords)
crackmapexec smb <lista_IPs> -u users.txt -p passwords.txt -d <dominio> --continue-on-success
```
```bash
# Tras encontrar una credencial válida: enumerar shares y permisos en todos los hosts
crackmapexec smb <lista_IPs> -u <user> -d <dominio> -p '<password>' --shares
```
| Flag / parte | Qué hace |
|---|---|
| `--continue-on-success` | no se detiene tras el primer login válido — sigue probando el resto de combinaciones/hosts |
| `--shares` | lista los recursos SMB compartidos de cada host y si la credencial tiene permiso de lectura/escritura sobre ellos |
| `(Pwn3d!)` en la salida | la credencial es **administrador local** en ese host |
| `STATUS_LOGON_FAILURE` | el **usuario existe** pero la contraseña es incorrecta (distinto de un error de usuario inválido) |

> [!warning] Detección / problemas
> - Versiones antiguas de CrackMapExec (ej. 5.4.0) pueden lanzar `The NETBIOS connection with the remote host is timed out` o no devolver salida contra un controlador de dominio — actualizar a una versión más reciente (5.4.1+) suele solucionarlo.
> - Un spraying agresivo (muchos intentos por usuario en poco tiempo) puede disparar **lockout** de cuentas de dominio — revisar la política de bloqueo (`net accounts` desde una sesión válida) antes de lanzar listas grandes.
> - Que un usuario no tenga permisos interesantes en ningún share visible con CME no significa que no los tenga en otro host no incluido en la lista inicial — repetir el spraying a medida que se descubren más hosts.

## Visto en
- Módulo: [[27 - Assembling the Pieces]]
- Relacionado: [[AD Password Spraying]] · [[Hydra]]
- Máquinas: [[]]
</content>
