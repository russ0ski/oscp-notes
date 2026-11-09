---
tipo: tecnica
fase: enumeracion
so: cloud
tags: [oscp, tecnica, enumeracion, cloud, aws]
fecha: 2026-10-09
---

# Enumerar IAM con AWS CLI (`get-account-authorization-details`)

> [!abstract] Qué es y cuándo usarla
> Con credenciales AWS comprometidas (acceso API), volcar de una sola llamada toda la configuración IAM relevante — users, groups, roles y sus policies — en vez de encadenar manualmente `list-user-policies`/`get-user-policy`/etc. para cada identidad. Es el primer comando a lanzar tras obtener credenciales comprometidas para mapear quién tiene qué permisos.

## Requisitos previos
- Credenciales AWS válidas con el permiso `iam:GetAccountAuthorizationDetails` (o `iam:Get*`).
- AWS CLI configurado, idealmente con un **profile** dedicado (`aws configure --profile <nombre>`) para no mezclar credenciales propias y comprometidas.

## Pasos
```bash
# Volcado completo, filtrando por tipo de elemento
aws --profile target iam get-account-authorization-details --filter User Group LocalManagedPolicy Role | tee auth-details.json

# Extraer solo lo necesario con JMESPath (--query)
aws --profile target iam get-account-authorization-details --filter User --query "UserDetailList[].UserName"
aws --profile target iam get-account-authorization-details --filter User --query "UserDetailList[?contains(UserName, 'admin')].UserName"
```
Revisar en el JSON resultante, para cada user/group/role de interés:
- Policies inline y managed directamente adjuntas.
- Policies heredadas por pertenencia a un **grupo**.
- Uso de wildcards (`"Action":"*"`, `"Resource":"*"`) o condiciones basadas en **tags** (ABAC) que puedan ampliar el alcance real más allá de lo evidente a simple vista.

> [!warning] Detección / problemas
> - No guardar la salida con `tee` obliga a repetir la llamada a la API cada vez que se necesita revisar algo — más ruido/CloudTrail.
> - Confundir `--filter` (limita qué pide la API, antes de la llamada) con `--query` (JMESPath, se aplica después sobre la respuesta ya recibida).
> - Un user "sin policies propias" puede seguir siendo altamente privilegiado por las policies de su grupo — nunca analizar un user de forma aislada.
> - Un `AccessDenied` al listar versiones de una policy concreta no implica que esa información sea inaccesible por otra vía: puede aparecer igualmente embebida en la salida de `get-account-authorization-details`.

## Visto en
- Módulo: [[25 - Enumerating AWS Cloud Infrastructure]]
- Relacionado: [[Pacu (AWS)]] · [[Descubrir Account ID AWS via Politica S3]]
- Máquinas: [[]]
