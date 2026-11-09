---
tipo: cheatsheet
tags: [oscp, cheatsheet, cloud, aws]
---

# Cheatsheet: AWS

> [!info] Comandos añadidos a medida que avanzo en los módulos.

## Recon DNS/WHOIS de infraestructura cloud
```bash
whois <dominio>
dig NS <dominio>                                            # *.awsdns-* => Route53/AWS
dig <subdominio.dominio>
dnsenum --dnsserver <IP_DNS> -f subdomains.txt <dominio>
```

## Buckets S3 (sin credenciales)
```bash
curl -s https://<bucket>.s3.amazonaws.com/
# NoSuchBucket -> no existe | AccessDenied -> existe, privado | <Contents> -> público y listable
cloud_enum -k <palabra_clave> --disable-azure --disable-gcp
```

## AWS CLI: profiles y recursos públicos
```bash
aws configure --profile <nombre>
aws --profile <nombre> ec2 describe-images --owners self --filters "Name=is-public,Values=true"
aws --profile <nombre> ec2 describe-snapshots --owner-ids <id> --filters "Name=status,Values=completed"
```

## Enumeración IAM con credenciales comprometidas
```bash
aws --profile target iam get-account-summary | tee account-summary.json
aws --profile target iam list-users | tee users.json
aws --profile target iam list-groups | tee groups.json
aws --profile target iam list-roles | tee roles.json
aws --profile target iam list-policies --scope Local --only-attached | tee policies.json
aws --profile target iam get-account-authorization-details --filter User Group LocalManagedPolicy Role | tee auth-details.json
aws --profile target iam list-attached-user-policies --user-name <user>
aws --profile target iam get-policy-version --policy-arn <arn> --version-id v1
```

## JMESPath (`--query`)
```bash
aws ... --query "UserDetailList[].UserName"
aws ... --query "UserDetailList[0].[UserName, Path, GroupList]"
aws ... --query "UserDetailList[?contains(UserName, 'admin')].UserName"
aws ... --query "{Users: UserDetailList[?Path=='/admin/'].UserName, Groups: GroupDetailList[?Path=='/admin/'].GroupName}"
```

## Pacu
```bash
sudo apt install pacu
pacu
# dentro: <nombre_sesion> ; import_keys <profile> ; ls ; help <modulo> ; run <modulo> ; services ; data <servicio>
```
