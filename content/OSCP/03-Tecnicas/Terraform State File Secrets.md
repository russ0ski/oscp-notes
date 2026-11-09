---
tipo: tecnica
fase: enumeracion
so: cloud
tags: [oscp, tecnica, enumeracion, cloud, aws]
fecha: 2026-10-09
---

# Terraform State File Secrets

> [!abstract] Qué es y cuándo usarla
> El fichero `terraform.tfstate` que Terraform usa para trackear el estado de la infraestructura gestionada guarda, con frecuencia, atributos sensibles de los recursos creados **en texto plano** (claves de acceso IAM, contraseñas, ARNs). Si ese fichero queda accesible (ej. en un bucket S3 mal configurado usado como backend remoto de Terraform), es un objetivo de muy alto valor: puede contener credenciales de cuentas con privilegios de administrador.

## Requisitos previos
- Credenciales (aunque sean de bajo privilegio) con permiso `s3:ListBucket`/`s3:GetObject` sobre el bucket que almacena el state (suele tener nombres tipo `tf-state-*`, `terraform-state-*`).

## Pasos
```bash
aws --profile <perfil> s3api list-buckets                  # buscar buckets con nombre tipo tf-state-*
aws --profile <perfil> s3 ls tf-state-<sufijo>
aws --profile <perfil> s3 cp s3://tf-state-<sufijo>/terraform.tfstate ./
cat terraform.tfstate | less                                  # o jq . terraform.tfstate para formatear
```
Revisar especialmente:
- La sección de **users** y sus `policy`/`AdministratorAccess` vs `ReadOnlyAccess` asociadas.
- La sección de **resources**, buscando atributos `id`/`secret`/`access_key` de cada recurso IAM creado.

```bash
# Con la credencial de mayor privilegio encontrada:
aws configure --profile=<nombre_admin>
aws --profile=<nombre_admin> iam list-attached-user-policies --user-name <nombre_admin>   # confirmar nivel de acceso
```

> [!warning] Detección / problemas
> - El state file puede ser grande — revisar por secciones (`users`, `resources`) en vez de leerlo entero de un tirón.
> - No asumir que el primer user listado es el más privilegiado — comparar las políticas de todos antes de elegir con cuál pivotar.
> - Si el bucket usa versionado, puede haber versiones antiguas del state con credenciales ya rotadas pero aún válidas para otros fines (reconocimiento histórico).

## Visto en
- Módulo: [[26 - Attacking AWS Cloud Infrastructure]]
- Relacionado: [[Enumerar IAM con AWS CLI]]
- Máquinas: [[]]
