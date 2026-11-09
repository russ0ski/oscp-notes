---
tipo: modulo
modulo: 25
estado: repasado
tags:
  - oscp
  - enumeracion
  - cloud
  - aws
fecha: 2026-10-09
---

# 25 - Enumerating AWS Cloud Infrastructure

> [!abstract] Resumen
> Reconocimiento y enumeración de infraestructura **AWS**, tanto desde fuera (sin credenciales) como desde dentro (con credenciales comprometidas).
> - **25.2 Descubrir infraestructura cloud**: DNS/WHOIS para identificar qué proveedor cloud (CSP) aloja un activo, `dnsenum` para fuerza bruta de subdominios, y revisión del tráfico con DevTools del navegador para localizar peticiones a dominios cloud (ej. buckets S3).
> - **25.3/25.4 Enumerar recursos sin credenciales**: descubrir y comprobar buckets **S3** por URL directa (distinguiendo `NoSuchBucket` / `AccessDenied` / listado abierto), automatizar con `cloud_enum`, y con credenciales propias de AWS CLI enumerar AMIs/snapshots compartidos públicamente, además de una técnica para **adivinar el Account ID** de un tercero abusando de condiciones de políticas IAM.
> - **25.5 Enumerar desde una cuenta comprometida**: con credenciales robadas, usar `aws iam get-account-authorization-details` para volcar users/groups/roles/policies, filtrar el JSON con **JMESPath** (`--query`), automatizar con **Pacu**, y extraer insights (rutas de escalado de privilegios) de los datos recolectados.
> Idea clave del examen: la enumeración cloud combina **OSINT clásico** (DNS/WHOIS) con **APIs del proveedor** (AWS CLI) — y, con credenciales comprometidas, el objetivo es mapear quién tiene qué permisos sobre qué recursos para encontrar una ruta a **AdministratorAccess**.

## Conceptos clave
- **CSP (Cloud Service Provider)**: AWS, Azure, GCP, etc. Cada uno tiene convenciones de nombrado de dominio propias para sus servicios (ej. S3 usa `<bucket>.s3.amazonaws.com` o `<bucket>.s3.<region>.amazonaws.com`; Azure Blob Storage usa `<cuenta>.blob.core.windows.net`). Identificar el CSP es el primer paso para saber qué herramientas/API usar.
- **DNS/WHOIS como recon cloud**: los registros DNS (en particular los nameservers, ej. `*.awsdns-*.com`/`*.awsdns-*.org`/`*.awsdns-*.net`/`*.awsdns-*.co.uk` para Route53) y las resoluciones reverse-IP permiten inferir qué recursos de un dominio están alojados en qué CSP, incluso sin acceso directo a la cuenta cloud.
- **`dnsenum`**: herramienta de fuerza bruta de subdominios vía DNS, útil para descubrir activos cloud-hosted que no aparecen en el dominio principal (ej. `s3-assets.empresa.com`, `api-staging.empresa.com`).
- **DevTools del navegador (pestaña Network)**: al navegar un sitio objetivo, inspeccionar las peticiones XHR/fetch y recursos estáticos revela URLs de servicios cloud subyacentes (ej. un bucket S3 sirviendo imágenes) que no son evidentes solo mirando el HTML.
- **Buckets S3**: contenedores de almacenamiento de objetos en AWS. Un bucket puede ser público (listable), privado (acceso denegado pero existente) o no existir. Se distingue por el mensaje de error XML devuelto al acceder por HTTP: `NoSuchBucket` (no existe), `AccessDenied` (existe pero sin permiso), listado XML de objetos (público y listable).
- **`cloud_enum`**: herramienta que automatiza la fuerza bruta de nombres de recursos cloud (buckets S3, contenedores Azure, buckets GCS) a partir de una lista de palabras clave, probando contra los patrones de nombrado de cada CSP.
- **AWS CLI y profiles**: `aws configure --profile <nombre>` permite guardar credenciales con un alias, usable luego con `--profile <nombre>` en cualquier subcomando sin pisar el profile `default`. Útil para manejar credenciales comprometidas de varias cuentas/objetivos a la vez.
- **Recursos compartidos públicamente**: AMIs (imágenes de máquina), snapshots de EBS (discos) y RDS (bases de datos) pueden compartirse como "públicos" por error de configuración. Se buscan con `aws ec2 describe-images`/`describe-snapshots` usando `--filters` (ej. `Name=is-public,Values=true`) incluso sin credenciales de la cuenta objetivo, usando una cuenta AWS propia.
- **Descubrir el Account ID de un tercero (técnica de política IAM)**: se puede crear una política IAM propia con una condición `StringLike` sobre `s3:ResourceAccount` y usarla para hacer una búsqueda binaria dígito a dígito del Account ID de otra cuenta, aprovechando que la política permite o deniega según si el patrón coincide.
- **Probing de ARNs vía `Principal`**: al intentar poner como `Principal` de una política de recursos (ej. de un bucket S3 propio) el ARN de un user/role de la cuenta objetivo, la respuesta de AWS (`Invalid principal in policy` vs. éxito) revela si ese ARN existe en la cuenta objetivo, permitiendo enumerar IAM users/roles de un tercero sin tener acceso a su cuenta.
- **`aws iam get-account-authorization-details`**: subcomando "todo en uno" que vuelca de golpe users, groups, roles y sus policies (inline y managed) en un único JSON, evitando tener que encadenar `list-user-policies`/`get-user-policy`/etc. manualmente. Requiere el permiso `iam:GetAccountAuthorizationDetails`. Se puede filtrar con `--filter User Group Role LocalManagedPolicy AWSManagedPolicy`.
- **JMESPath (`--query`)**: lenguaje de queries para JSON que el AWS CLI soporta de forma nativa vía `--query`. A diferencia de `--filter` (que limita qué recursos pide la API), `--query` se aplica sobre la respuesta ya recibida, permitiendo extraer campos concretos (`UserDetailList[].UserName`), filtrar por condición (`[?contains(UserName, 'admin')]`) o construir objetos JSON nuevos combinando varias rutas.
- **Pacu**: framework de post-explotación/enumeración para AWS (análogo a Metasploit mono-cloud). Organiza el trabajo en `sessions`, importa credenciales con `import_keys <profile>`, y expone módulos categorizados (`[Category: ENUM]`, etc.) ejecutables con `run <modulo>`; los datos recolectados se guardan en una base de datos local consultable con `data <servicio>`.
- **Tags como pista de ABAC**: las AWS Managed/Custom Policies pueden usar Attribute-Based Access Control, concediendo permisos a cualquier recurso que tenga un `tag` concreto (clave/valor). Un user sin policies propias pero con un tag puede heredar permisos completos vía una policy de su grupo que referencia ese tag — hay que revisar tags de users/groups, no solo policies directas.
- **Extraer insights de la enumeración**: el objetivo final no es solo listar recursos sino encontrar la **ruta de escalado de privilegios** — ej. un user sin policies propias pero miembro de un grupo con `AdministratorAccess`, o una policy con wildcard `"Action":"*"`/`"Resource":"*"` combinada con pertenencia a un grupo con `iam:CreateAccessKey` sobre otros users del mismo grupo.

> [!tip] Idea para el examen
> Flujo recomendado: sin credenciales → DNS/WHOIS + `dnsenum` + DevTools para identificar el CSP y activos cloud → probar buckets S3 por URL directa / `cloud_enum` → si hay credenciales AWS propias, buscar AMIs/snapshots públicos y considerar la técnica de Account ID vía política S3. Con credenciales comprometidas → `aws iam get-account-authorization-details` (con `--filter`) como primer volcado completo → `--query` (JMESPath) para no procesar manualmente JSON enorme → cruzar policies de user + policies del grupo (ojo a tags y wildcards) → Pacu si se quiere automatizar/organizar varias sesiones.

---

## 25.2 Descubriendo infraestructura cloud

```bash
whois <dominio>                                   # registrar, nameservers (pista de CSP)
dig NS <dominio>                                    # nameservers directamente
dig <subdominio.dominio>                              # resolución, IP y posible CNAME a servicio cloud
dnsenum --dnsserver <IP_DNS> -f subdomains.txt <dominio>    # fuerza bruta de subdominios
```
| Flag / parte | Qué hace |
|---|---|
| `dig NS <dominio>` | muestra los nameservers — patrones como `*.awsdns-*.{com,net,org,co.uk}` indican Route53 (AWS) |
| `dnsenum -f subdomains.txt` | prueba cada palabra del diccionario como subdominio, revelando activos cloud no listados en el sitio principal |

En el navegador (DevTools → pestaña **Network**), recargar la página objetivo y filtrar por `xhr`/`fetch`/`img` para localizar peticiones a dominios de CSP (ej. `*.s3.amazonaws.com`, `*.blob.core.windows.net`).

| CSP | Convención de nombrado típica |
|---|---|
| AWS (S3) | `<bucket>.s3.amazonaws.com` · `<bucket>.s3.<region>.amazonaws.com` · `s3.amazonaws.com/<bucket>` |
| Azure (Blob Storage) | `<cuenta>.blob.core.windows.net/<contenedor>` |
| GCP (Cloud Storage) | `storage.googleapis.com/<bucket>` · `<bucket>.storage.googleapis.com` |

---

## 25.3 Enumerando buckets S3 sin credenciales

```bash
curl -s https://<bucket>.s3.amazonaws.com/            # acceso directo por HTTP
```
La respuesta XML distingue tres casos:
- `<Code>NoSuchBucket</Code>` → el bucket no existe (ese nombre está libre).
- `<Code>AccessDenied</Code>` → el bucket **existe** pero no es listable (requiere permisos).
- Listado XML de `<Contents>` → el bucket es **público y listable**: se pueden descargar sus objetos directamente por URL.

```bash
cloud_enum -k <palabra_clave> --disable-azure --disable-gcp    # fuerza bruta de buckets S3 con una palabra clave
```
| Flag / parte | Qué hace |
|---|---|
| `-k <palabra_clave>` | base del nombre a probar contra los patrones de nombrado de cada CSP (ej. `empresa`, `empresa-backups`, `empresa-dev`) |
| `--disable-azure --disable-gcp` | limita la búsqueda a AWS si ya se sabe que el CSP es AWS |

---

## 25.4 Enumerando recursos adicionales con AWS CLI

```bash
aws configure --profile <nombre>                       # guarda credenciales propias bajo un alias
aws --profile <nombre> ec2 describe-images --owners self --filters "Name=is-public,Values=true"
aws --profile <nombre> ec2 describe-snapshots --owner-ids <id_propietario> --filters "Name=status,Values=completed"
```
| Flag / parte | Qué hace |
|---|---|
| `--profile <nombre>` | usa las credenciales guardadas bajo ese alias en vez del profile `default` |
| `--filters "Name=is-public,Values=true"` | limita el resultado a AMIs/snapshots marcados como públicos por error de configuración |

**Descubrir el Account ID de un tercero**: crear, en una cuenta AWS propia, una política IAM con una condición `StringLike` sobre `s3:ResourceAccount` (ej. `"123456789???"`) y probar binariamente dígito a dígito si la política permite o deniega el acceso — permite reconstruir el Account ID completo sin acceso a la cuenta objetivo.

**Probing de IAM users/roles de un tercero vía `Principal`**: en una policy de recursos propia (ej. de un bucket S3 propio), poner como `Principal` el ARN supuesto de un user/role de la cuenta objetivo (`arn:aws:iam::<account_id>:user/<nombre>`) e intentar aplicarla — un error `Invalid principal in policy` indica que ese ARN **no existe**; si la policy se guarda sin error, el ARN **sí existe** en esa cuenta.

---

## 25.5 Enumerando desde una cuenta comprometida (IAM)

### Permisos de partida y volcado inicial
```bash
aws --profile target iam get-policy-version --policy-arn <arn_policy_adjunta> --version-id v1 | grep -E "iam|Get\*|List\*"
aws --profile target iam help | grep -E "list-|get-|generate-"           # subcomandos IAM disponibles
aws --profile target iam get-account-summary | tee account-summary.json     # conteo de users/groups/roles/policies, MFA
aws --profile target iam list-users | tee users.json
aws --profile target iam list-groups | tee groups.json
aws --profile target iam list-roles | tee roles.json
aws --profile target iam list-policies --scope Local --only-attached | tee policies.json
```
| Flag / parte | Qué hace |
|---|---|
| `| tee <fichero>.json` | guarda la salida en disco (además de mostrarla) para procesarla después sin repetir llamadas a la API y reducir ruido/logging |
| `--scope Local --only-attached` | lista solo policies custom de la cuenta (no las AWS Managed) que están realmente adjuntas a algo |

> [!warning] `get-account-summary` con `"MFADevicesInUse": 0` y `"AccountMFAEnabled": 0` es una señal de mala praxis (sin MFA) que facilita further exploitation, aunque no es en sí un vector de privesc.

### Volcado completo con `get-account-authorization-details`
```bash
aws --profile target iam get-account-authorization-details --filter User Group LocalManagedPolicy Role
```
| Flag / parte | Qué hace |
|---|---|
| `--filter User Group LocalManagedPolicy Role` | limita qué tipos de elementos incluye la respuesta (`User`, `Role`, `Group`, `LocalManagedPolicy`, `AWSManagedPolicy`) |

> [!tip] Requiere el permiso `iam:GetAccountAuthorizationDetails` (o `iam:Get*`). Devuelve de una sola vez `UserDetailList`, `GroupDetailList`, `RoleDetailList` y `Policies`, evitando encadenar `list-*-policies` + `get-*-policy` uno a uno por cada identidad.

### Filtrando con JMESPath (`--query`)
```bash
aws --profile target iam get-account-authorization-details --filter User --query "UserDetailList[].UserName"
aws --profile target iam get-account-authorization-details --filter User --query "UserDetailList[0].[UserName, Path, GroupList]"
aws --profile target iam get-account-authorization-details --filter User --query "UserDetailList[?contains(UserName, 'admin')].{Name: UserName}"
aws --profile target iam get-account-authorization-details --filter User Group --query "{Users: UserDetailList[?Path=='/admin/'].UserName, Groups: GroupDetailList[?Path=='/admin/'].GroupName}"
```
| Flag / parte | Qué hace |
|---|---|
| `--filter` | se aplica **antes** de la petición — reduce qué pide la API |
| `--query` | se aplica **después**, sobre el JSON ya recibido — permite extraer campos, filtrar (`[?condicion]`) o construir objetos nuevos combinando varias rutas |
| `[0]` | selecciona el primer elemento de un array |
| `[?contains(UserName, 'admin')]` | filtra elementos cuyo campo cumple la condición |

### Automatizando con Pacu
```bash
sudo apt install pacu
pacu                                      # modo interactivo
# dentro de pacu:
#   <nombre_sesion>                        # crear/nombrar sesión
# import_keys <profile_aws_cli>            # importa credenciales ya guardadas en el AWS CLI
# ls                                       # lista módulos disponibles ([Category: ENUM], etc.)
# help iam__enum_users_roles_policies_groups
# run iam__enum_users_roles_policies_groups --users --roles --policies --groups
# services                                 # servicios con datos recolectados
# data IAM                                 # consulta los datos de IAM guardados en la BBDD local de Pacu
```
| Flag / parte | Qué hace |
|---|---|
| `import_keys <profile>` | importa las credenciales ya configuradas en `~/.aws/credentials` bajo ese profile, sin tener que volver a pegarlas |
| `run <modulo>` | ejecuta el módulo de enumeración/explotación elegido; los resultados se guardan en la BBDD local de Pacu, no solo en pantalla |
| `data <servicio>` | muestra todo lo recolectado para ese servicio (ej. `IAM`) combinando los datos de varios módulos ejecutados |

> [!warning] Monitorizar siempre CloudTrail (o asumir que está activo) antes de lanzar módulos de Pacu — cada llamada API queda registrada, y Pacu puede generar muchas llamadas de golpe.

### Extrayendo insights (ejemplo de ruta a privesc)
Analizando un user `admin-alice` sin policies propias pero miembro del grupo `admin` (con `AdministratorAccess` adjunta) y del grupo `amethyst_admin` (policy custom con wildcards `"Action":"*"` sobre recursos **tagueados**), se identifican dos rutas a privilegio total:
1. Conseguir ejecutar acciones como `admin-alice` directamente (ej. generar nuevas access keys si la policy lo permite).
2. Encontrar otro miembro del mismo grupo con permisos de escritura sobre IAM (ej. `iam:AddUserToGroup`/`iam:CreateAccessKey`) para pivotar sobre otro user del grupo con más acceso.

```bash
aws --profile target iam get-account-authorization-details --filter User Group --query "UserDetailList[?UserName=='admin-alice']"
aws --profile target iam get-account-authorization-details --filter User Group --query "GroupDetailList[?GroupName=='admin' || GroupName=='amethyst_admin']"
aws --profile target iam get-account-authorization-details --filter LocalManagedPolicy --query "Policies[?PolicyName=='amethyst_admin'].PolicyVersionList"
```

---

> [!warning] Errores comunes / cosas que fallan
> - Confundir `--filter` (limita qué pide la API, va **antes** de la llamada) con `--query` (JMESPath sobre la respuesta ya recibida).
> - No guardar la salida de `get-account-authorization-details` en fichero (`tee`) obliga a repetir llamadas a la API — más ruido en CloudTrail.
> - Dar por hecho que un user "sin policies" no tiene privilegios — revisar siempre las policies de sus **grupos**, no solo las suyas directas.
> - Un `AccessDenied` al listar policy versions no significa que la policy sea inaccesible: a veces el mismo documento aparece embebido en la salida de `get-account-authorization-details` aunque `list-policy-versions` esté explícitamente denegado.
> - Lanzar módulos de Pacu sin haber comprobado antes el estado de CloudTrail/logging puede generar alertas innecesarias en un entorno real.
> - Dar por buena la lista de atributos que recolecta un módulo de Pacu: cada módulo solo guarda lo que su autor consideró relevante — puede faltar algo que sí aparece con el subcomando AWS CLI equivalente.

## Ejemplo práctico
> [!example] Cadena de enumeración AWS (lab "enumlab")
> 1. Sin credenciales: `dig NS` revela nameservers `*.awsdns-*` → el objetivo usa Route53/AWS. DevTools muestra una petición a `assets.s3.amazonaws.com` → se confirma un bucket S3.
> 2. `curl https://assets.s3.amazonaws.com/` devuelve un listado XML completo → bucket público y listable, se descargan sus objetos directamente.
> 3. Con credenciales AWS propias, `cloud_enum -k empresa` encuentra además `empresa-backups` (existe pero `AccessDenied`) y `empresa-xyz123` (no existe, `NoSuchBucket`).
> 4. Credenciales comprometidas del IAM user `clouddesk-plove` (grupo `support`, policy `SupportUser`): `iam get-policy-version` revela permisos `iam:Get*`/`iam:List*` → se puede enumerar IAM aunque no modificarlo.
> 5. `aws iam get-account-authorization-details --filter User Group LocalManagedPolicy Role` vuelca toda la configuración IAM; se guarda en ficheros `.json` con `tee`.
> 6. Con `--query "UserDetailList[?contains(UserName,'admin')].UserName"` se localizan rápidamente varios users `admin-*` sin tener que leer el JSON entero a mano.
> 7. Se detecta que `admin-alice` no tiene policies propias pero es miembro del grupo `admin` (`AdministratorAccess`) y de `amethyst_admin` (policy con wildcards sobre recursos tagueados `Project=amethyst`) → ruta directa a privilegio total si se consigue ejecutar acciones como ella o como otro miembro del grupo.
> 8. Se repite la misma enumeración con Pacu (`import_keys target` → `run iam__enum_users_roles_policies_groups` → `data IAM`) para confirmar los mismos hallazgos de forma automatizada.

## Relacionado
- Técnicas: [[Enumeracion de Buckets S3]] · [[Descubrir Account ID AWS via Politica S3]] · [[Pacu (AWS)]] · [[Enumerar IAM con AWS CLI]]
- Cheatsheet: [[AWS]]
- Módulo previo: [[24 - Lateral Movement in Active Directory]]
- Relacionado: [[SearchSploit]] (metodología de búsqueda similar), [[Metasploit]] (framework de post-explotación análogo en estructura a Pacu)
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Repasar las convenciones de nombrado de Azure/GCP con el mismo nivel de detalle que AWS, por si aparece un objetivo multi-cloud en el examen.
> - Profundizar en la técnica de descubrir el Account ID vía condición de política S3 — repetir el proceso paso a paso para interiorizarlo antes del examen.
> - Ver más módulos de Pacu fuera de IAM (S3, EC2, Secrets Manager) para tener un catálogo mental más amplio.
