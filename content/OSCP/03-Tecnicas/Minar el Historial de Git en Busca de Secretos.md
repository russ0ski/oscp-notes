---
tipo: tecnica
fase: enumeracion
so: linux
tags: [oscp, tecnica, enumeracion, privesc]
fecha: 2026-10-09
---

# Minar el Historial de Git en Busca de Secretos

> [!abstract] Qué es y cuándo usarla
> Borrar un fichero de un repositorio Git (`git rm` + commit) o editar una línea con un secreto no elimina ese contenido del **historial** de commits — sigue siendo recuperable mientras el objeto exista en `.git/`. Cuando se tiene acceso de lectura a un repositorio (local tras una shell, o descargado de un bucket/servidor mal configurado), revisar el historial commit a commit es una forma habitual de encontrar credenciales que el equipo de desarrollo creía ya eliminadas.

## Requisitos previos
- Acceso de lectura al directorio `.git/` de un repositorio (shell local, o repo descargado completo).

## Pasos
```bash
cd /ruta/al/repo
git log --all --oneline                      # ver todos los commits, incluidos los de ramas no activas
git show <hash_commit>                        # diff exacto de un commit concreto
git log -p -- <ruta/fichero>                   # historial completo de cambios de UN fichero concreto
```
Buscar específicamente commits con mensajes tipo "remove credentials", "fix", "cleanup" — suelen ser los que borran el secreto del `HEAD` sin eliminarlo del historial.

```bash
# Herramientas automáticas como complemento, NUNCA como sustituto de la revisión manual
sudo apt install -y gitleaks
gitleaks detect
```

Si se encuentra un header de autenticación o credencial codificada:
```bash
echo "<base64>" | base64 --decode
```

> [!warning] Detección / problemas
> - `gitleaks detect` puede no encontrar nada (busca por regex/entropía) aunque el secreto esté ahí — siempre complementar con revisión manual de `git log`/`git show`.
> - **Nunca** usar `git checkout <hash>` en un repositorio que sirve contenido en producción (ej. un sitio WordPress activo) — puede dejar el sitio en un estado roto o inconsistente. `git show` es de solo lectura y no toca el working tree.
> - El secreto puede estar en un fichero que ya no existe en absoluto en el `HEAD` — buscar también en `git log --all --diff-filter=D` (commits que borraron ficheros).

## Visto en
- Módulo: [[27 - Assembling the Pieces]], [[26 - Attacking AWS Cloud Infrastructure]] (mismo concepto, aplicado a un bucket S3 con `.git/`)
- Relacionado: [[Abusing Sudo (GTFOBins)]]
- Máquinas: [[]]
</content>
