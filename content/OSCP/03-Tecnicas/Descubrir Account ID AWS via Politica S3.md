---
tipo: tecnica
fase: enumeracion
so: cloud
tags: [oscp, tecnica, enumeracion, cloud, aws]
fecha: 2026-10-09
---

# Descubrir Account ID AWS vía política S3

> [!abstract] Qué es y cuándo usarla
> Técnica para averiguar el **Account ID** (número de 12 dígitos) de una cuenta AWS de un tercero, sin tener acceso a ella, abusando de una condición `StringLike` sobre `s3:ResourceAccount` en una política IAM creada en una cuenta AWS **propia**. Al probar distintos patrones de dígitos y observar si la política permite o deniega, se reconstruye el Account ID por búsqueda binaria.

## Requisitos previos
- Una cuenta AWS propia con permisos para crear/editar políticas IAM.
- Saber (o sospechar) algún recurso S3 del objetivo cuyo ARN incluya el Account ID a adivinar.

## Pasos
1. Crear una política IAM propia con una condición `StringLike` sobre `s3:ResourceAccount`, probando un patrón con comodines (ej. `"123456789???"`).
2. Intentar una acción S3 que dispare la evaluación de esa condición contra el recurso del objetivo.
3. Si la política permite la acción, el patrón coincide con el Account ID real; si la deniega, ajustar el patrón.
4. Repetir el proceso dígito a dígito (búsqueda binaria sobre el espacio de dígitos restante) hasta reconstruir el Account ID completo.

> [!warning] Detección / problemas
> - Es una técnica lenta si se hace manualmente dígito a dígito; conviene scriptearla si se va a repetir varias veces.
> - Depende de poder referenciar un recurso S3 del objetivo (aunque sea de forma indirecta) desde la política propia.
> - Cada intento es una llamada/evaluación de política — en un entorno con monitorización activa esto puede ser detectable si se hace a gran volumen.

## Visto en
- Módulo: [[25 - Enumerating AWS Cloud Infrastructure]]
- Relacionado: [[Enumeracion de Buckets S3]] · [[Enumerar IAM con AWS CLI]]
- Máquinas: [[]]
