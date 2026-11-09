---
tipo: tecnica
fase: enumeracion
so: cloud
tags: [oscp, tecnica, enumeracion, cloud, aws]
fecha: 2026-10-09
---

# Enumeración de Buckets S3

> [!abstract] Qué es y cuándo usarla
> Técnica para descubrir y evaluar buckets **S3** de AWS sin necesidad de credenciales: se prueba el nombre directamente por URL y se interpreta el error XML devuelto para saber si el bucket existe, es privado o es público y listable.

## Requisitos previos
- Posibles nombres de bucket candidatos (OSINT, DNS/WHOIS, DevTools del navegador, o lista de palabras clave relacionadas con la empresa objetivo).
- `curl` o un navegador; opcionalmente `cloud_enum` instalado para automatizar la fuerza bruta.

## Pasos
```bash
curl -s https://<bucket>.s3.amazonaws.com/
```
Interpretar la respuesta XML:
- `<Code>NoSuchBucket</Code>` → el nombre no está en uso.
- `<Code>AccessDenied</Code>` → el bucket existe pero no es listable sin permisos.
- `<ListBucketResult>` con `<Contents>` → bucket público y listable; los objetos se pueden descargar directamente por `https://<bucket>.s3.amazonaws.com/<objeto>`.

Automatizar la búsqueda de nombres candidatos:
```bash
cloud_enum -k <palabra_clave> --disable-azure --disable-gcp
```

> [!warning] Detección / problemas
> - Un bucket `AccessDenied` no es un callejón sin salida: revisar si algún objeto específico dentro de él es accesible aunque el listado general no lo sea.
> - Los nombres de bucket son globales en todo AWS — un nombre "obvio" puede estar ya ocupado por otra organización sin relación con el objetivo.
> - `cloud_enum` genera muchas peticiones HTTP en poco tiempo; en un entorno con monitorización real esto puede generar alertas.

## Visto en
- Módulo: [[25 - Enumerating AWS Cloud Infrastructure]]
- Relacionado: [[Descubrir Account ID AWS via Politica S3]]
- Máquinas: [[]]
