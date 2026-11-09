---
tipo: tecnica
fase: explotacion
so: multi
tags: [oscp, tecnica, explotacion, password]
fecha: 2026-10-05
---

# Password Cracking (Hashcat y John)

> [!abstract] Qué es y cuándo usarla
> Crackeo **offline** de hashes robados: hashear candidatos y comparar con el hash objetivo. No toca la red ni bloquea cuentas. **Hashcat** (GPU, rápido) vs **John the Ripper** (CPU, maneja formatos que Hashcat no, ej. claves SSH `aes-256-ctr`). Úsala tras extraer un hash (BBDD, SAM, KeePass, clave SSH...).

## Metodología (5 pasos)
1. **Extraer** el hash.
2. **Formatear** (scripts `*2john`; quitar el prefijo `filename:` del hash).
3. **Calcular tiempo** = keyspace / hash rate (si > vida útil, cambia de enfoque).
4. **Preparar wordlist** (reglas, password policy, leaks).
5. **Atacar**. Cuidado con espacios/newlines; confirma el **tipo** de hash (`hashid`, `hash-identifier`).

## Teoría del tiempo de crackeo
```bash
echo -n "secret1" | sha256sum                  # -n = sin newline (si no, cambia el hash)
echo -n "abc...0..9" | wc -c                    # tamaño del charset (62)
python3 -c "print(62**8)"                       # keyspace = charset^longitud
hashcat -b                                      # benchmark: hash rate por algoritmo (MH/s)
python3 -c "print(218340105584896 / 9276300000)"  # keyspace / hashrate = segundos
```
> **Longitud** aumenta el tiempo exponencialmente; **complejidad** (charset) solo polinomial → políticas de **longitud** protegen más.

## Reglas (mutar wordlist)
```bash
echo \$1 > demo.rule                            # $x append, ^x prepend
hashcat -r demo.rule --stdout demo.txt          # --stdout = solo muestra mutaciones (debug)
ls /usr/share/hashcat/rules/                    # best66, rockyou-30000, dive, leetspeak...
```
- Funciones en **una línea** = encadenadas izq→der (`c $1 $!` → `Password1!`).
- Funciones en **líneas distintas** = reglas independientes (varias salidas por password).

| Rule | Efecto |
|---|---|
| `c` | Capitaliza 1ª letra |
| `$X` / `^X` | Append / prepend de `X` |

## Atacar (modos y ejemplos)
```bash
hashcat --help | grep -i <algoritmo>            # buscar el modo (-m)
# MD5 = 0 ; NTLM = 1000 ; NetNTLMv2 = 5600 ; KeePass = 13400 ; SSH($6$)=22921
hashcat -m 0 crackme.txt rockyou.txt -r demo3.rule --force       # --force si no hay GPU

# KeePass: formatear y borrar "Database:" del hash
keepass2john Database.kdbx > keepass.hash
hashcat -m 13400 keepass.hash rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force

# Clave SSH: Hashcat 22921 falla con aes-256-ctr -> usar John
ssh2john id_rsa > ssh.hash                      # $6$ = SHA-512
# añadir [List.Rules:sshRules] + funciones a /etc/john/john.conf
john --wordlist=ssh.passwords --rules=sshRules ssh.hash
```

| Herramienta | Nota |
|---|---|
| `hashcat -m <modo> <hash> <wordlist> -r <rule>` | GPU; `--force` ignora warnings sin GPU |
| `john --wordlist=.. --rules=NAME <hash>` | CPU; reglas se definen en `/etc/john/john.conf` |
| `keepass2john` / `ssh2john` | Suite JtR: formatean `.kdbx` / claves privadas |

> [!warning] Detección / problemas
> - **"Token length exception"** en Hashcat con clave SSH → cifrada con `aes-256-ctr` no soportado por el modo 22921 → **usa John**.
> - Quita el prefijo `filename:` que añaden los `*2john` (salvo que quieras el username, p.ej. BBDD).
> - Un espacio o newline de más en el hash arruina el crackeo. Verifica el **tipo** antes.
> - Error "Not enough allocatable device memory" → sube la RAM de la VM (4 GB bastan).

## Visto en
- Módulo: [[16 - Password Attacks]]
- Relacionado: [[Mimikatz]] · [[Net-NTLMv2 (Responder y Relay)]] · [[Hydra]]
- Máquinas: [[]]
