---
tipo: tecnica
fase: enumeracion
so: cloud
tags: [oscp, tecnica, enumeracion, cloud, aws]
fecha: 2026-10-09
---

# Pacu (AWS)

> [!abstract] Qué es y cuándo usarla
> Framework de post-explotación/enumeración para AWS (análogo conceptual a Metasploit pero centrado en cloud). Organiza el trabajo en sesiones, importa credenciales ya configuradas en el AWS CLI y ejecuta módulos categorizados (enumeración IAM, S3, EC2, etc.) guardando los resultados en una base de datos local consultable. Útil cuando se quiere automatizar y organizar la enumeración de una cuenta comprometida en vez de ir lanzando comandos sueltos de AWS CLI.

## Requisitos previos
- `sudo apt install pacu` (disponible en los repos de Kali).
- Credenciales AWS comprometidas ya configuradas como un profile del AWS CLI (`aws configure --profile <nombre>`).

## Pasos
```bash
pacu                              # arranca en modo interactivo
```
Dentro de la consola de Pacu:
```
<nombre_sesion>                    # crea/nombra una sesión (organiza el trabajo por objetivo)
import_keys <profile_aws_cli>       # importa las credenciales ya guardadas en ~/.aws/credentials
ls                                  # lista todos los módulos disponibles, agrupados por categoría
help <modulo>                        # muestra uso y argumentos de un módulo concreto
run <modulo> [argumentos]             # ejecuta el módulo elegido
services                             # muestra qué servicios tienen datos recolectados
data <servicio>                       # consulta los datos guardados para ese servicio (ej. IAM)
```
Ejemplo de enumeración IAM:
```
run iam__enum_users_roles_policies_groups --users --roles --policies --groups
data IAM
```

> [!warning] Detección / problemas
> - Monitorizar (o asumir activo) **CloudTrail** antes de lanzar módulos — cada acción de Pacu genera llamadas API reales que quedan registradas.
> - Cada módulo solo recolecta los atributos que su autor consideró relevantes — puede faltar algún campo que sí se obtiene con el subcomando AWS CLI equivalente; no asumir que Pacu es exhaustivo al 100%.
> - Las credenciales importadas quedan guardadas en la base de datos local de Pacu (`~/.local/share/pacu/sqlite.db`) — limpiarla en entornos sensibles.

## Visto en
- Módulo: [[25 - Enumerating AWS Cloud Infrastructure]]
- Relacionado: [[Enumerar IAM con AWS CLI]]
- Máquinas: [[]]
