---
tipo: modulo
modulo: 6
estado: repasado
tags:
  - oscp
  - enumeracion
  - linux
  - windows
fecha: 2026-09-26
---

# 06 - Information Gathering

> [!abstract] Resumen
> Cómo mapear la superficie de ataque de un objetivo de forma **pasiva** (OSINT, sin tocar al target) y **activa** (interactuando directamente con sus servicios). Cubre whois, Google dorks, Netcraft, repos públicos, Shodan, cabeceras/SSL, y enumeración activa de DNS, SMB, SMTP y SNMP. También cómo apoyarse en LLMs y en técnicas *Living off the Land* (LOLBAS) desde Windows.
> El *information gathering* (o enumeración) **no termina** en el recon inicial: es cíclico y se repite tras cada foothold o movimiento lateral.

## Conceptos clave
- **Fases de un pentest**: Scoping → Information Gathering → Vuln Detection → Foothold → Privesc → Movimiento Lateral → Reporting → Remediación.
- **Pasivo vs activo**: pasivo = casi sin interacción (huella baja); activo = probamos la infra directamente (más ruido, más detalle).
- **Dos escuelas del "pasivo"**: estricta (nunca tocar el target, solo terceros) vs laxa (interactuar como un usuario normal, sin testear vulns). El curso usa la laxa.
- **Cíclico**: cada resultado alimenta el siguiente scan. Todo va a las notas de recon.
- **LOLBAS** (Living Off the Land Binaries And Scripts): binarios nativos de Windows (`nslookup`, `net`, `whoami`, `Test-NetConnection`...) reutilizados para enumerar sin instalar herramientas ni levantar alertas. Útil en escenarios *assumed breach*.

> [!tip] Idea para el examen
> El detalle más pequeño puede ser el pivote más grande. Documenta **todo** (nombres, emails, subdominios, versiones) aunque no parezca útil: correlaciona más tarde. En el examen prioriza: enumeración de servicios completa antes de lanzarte a explotar.

---

## Passive Information Gathering (OSINT)

### Whois
Protocolo sobre **TCP/43** que consulta bases de datos públicas de registro de dominios. Devuelve name servers, registrar, y contactos (registrant/admin/tech: nombre, org, email, teléfono), fechas de creación/expiración y estado del dominio.

- **Forward lookup**: partimos de un dominio → quién lo posee.
- **Reverse lookup**: partimos de una IP → a qué entidad/ISP pertenece (revela el CIDR, ej. `38.0.0.0/8` → PSINet).

En el lab de MegaCorp One, el whois reveló a *Alan Grofield* (IT & Security Director) como registrant/admin/tech, y los name servers `NS1-3.MEGACORPONE.COM`.

### Google Hacking (dorks)
Uso de operadores de búsqueda para refinar resultados y encontrar info sensible o mal configurada. Proceso iterativo: empezar amplio y estrechar con operadores. Ver nota: [[Google Dorks]].

> [!example] Hallazgo del lab
> `site:megacorpone.com filetype:txt` encontró el `robots.txt`, que revelaba `/nanites.php` — una página oculta a la búsqueda normal pero listada como *Allow* en la política.

La **Google Hacking Database (GHDB)** de Exploit-DB y el portal **DorkSearch** ofrecen dorks prefabricados.

### Netcraft
Portal web que hace OSINT pasivo: tecnologías de un sitio y hosts que comparten netblock. Su *site report* muestra registro, historia y **site technology** (subdominios + stack). El servicio de DNS search se descontinuó en 2024; alternativa en vivo: `wappalyzer.com/lookup/<dominio>`.

### Código open-source (GitHub, GitLab, Gist, SourceForge)
El código público revela lenguajes/frameworks y, a veces, **credenciales commiteadas por error**. La búsqueda de GitHub soporta operadores; para buscar en todos los repos públicos hace falta cuenta.

> [!example] Hallazgo del lab
> Buscando `owner:megacorpone path:users` apareció un único fichero `xampp.users` con un hash apache APR1:
> ```
> trivera:$apr1$A0vSKwao$GV3sgGAj53j.c3GkS4oUC0
> ```
> Un usuario (`trivera`) + hash crackeable, oro para la fase activa.

Para repos grandes, herramientas automáticas: **Gitrob**, **Gitleaks** (detección por regex o por **entropía** — strings muy aleatorias = probable secreto). Gitleaks encontró en el lab una *AWS Client ID* (regla `AWS Client ID`, tags `key, AWS`). Casi todas requieren *personal access token* por el rate-limit de la API.

### Shodan
Buscador de **dispositivos** conectados a internet (servidores, routers, IoT), no de contenido web. Muestra IPs, servicios, banners y versiones. Búsqueda: `hostname:megacorpone.com`. Filtrando por *Top Ports* (ej. SSH) vemos la versión exacta de OpenSSH por host y, al abrir una IP, un resumen con puertos, tecnologías y **vulnerabilidades publicadas**. Requiere cuenta (gratis, limitada).

### Security headers y SSL/TLS
Sitios de terceros que analizan la postura de seguridad (la línea pasivo/activo se difumina, pero el scan lo lanza un tercero):
- **SecurityHeaders.com**: analiza cabeceras HTTP. En el lab faltaban `Content-Security-Policy` y `X-Frame-Options` → indicio de admins que no aplican *hardening*.
- **Qualys SSL Labs (SSL Server Test)**: analiza config TLS y detecta vulns (Poodle, Heartbleed). En el lab el servidor aún soportaba TLS 1.0/1.1 y la suite `TLS_DHE_RSA_WITH_AES_256_CBC_SHA` (legacy, insegura).

> [!tip] LLMs para OSINT
> Un LLM (ej. ChatGPT) puede lanzar `whois`, listar estructura y empleados de una empresa, generar sets de Google dorks a medida y resumir el stack tecnológico. **Cautela**: alucina, usa datos desactualizados, transmite lo que le pegas a terceros — verifica siempre con fuentes fiables y confirma con el cliente que entra en scope.

---

## Active Information Gathering

Interacción directa con los servicios del objetivo. Aquí empezamos a generar tráfico y huella. Desde Kali con herramientas dedicadas o, en *assumed breach*, desde una workstation Windows con **LOLBAS**.

### DNS enumeration
DNS traduce nombres a IPs. Registros útiles: **NS** (servidores autoritativos), **A** (IPv4), **AAAA** (IPv6), **MX** (correo, con prioridad), **PTR** (reverse), **CNAME** (alias), **TXT** (arbitrario, ownership...). Es un objetivo muy jugoso. Ver nota: [[DNS Enumeration]].

- `host` para consultas puntuales; `-t` fija el tipo.
- **Brute-force forward**: iterar hostnames de un wordlist y ver cuáles resuelven.
- **Brute-force reverse (PTR)**: barrer un rango de IPs contiguo (los subdominios suelen agruparse, ej. `167.114.21.X`).
- Automatización: **dnsrecon**, **dnsenum** (que además intenta *zone transfer*), **gobuster dns**.
- Windows/LOLBAS: **nslookup** (`-type=TXT`).

### Port scanning (teoría)
Inspeccionar puertos TCP/UDP para descubrir servicios y vectores. **Ilegal sin permiso escrito** fuera del lab; puede tumbar servicios o disparar IDS/IPS. Metodología: empezar por puertos clave (80/443), lanzar full scan en background y estrechar en scans posteriores.

- **TCP CONNECT scan**: completa el three-way handshake (SYN → SYN-ACK → ACK). Puerto abierto = handshake completo.
- **UDP scan**: sin handshake. Se envía paquete UDP vacío; si el puerto está **cerrado**, el target responde **ICMP port unreachable**; si está abierto, normalmente **no responde** (la app decide). Poco fiable: un firewall que dropea ICMP hace que un puerto cerrado parezca abierto.

> [!example] Netcat como escáner rudimentario (capturas Wireshark del lab)
> TCP a 3388-3390: solo el **3389** devolvió **SYN-ACK** (abierto); 3388 y 3390 devolvieron **RST-ACK** (rechazados). Netcat cerró con FIN-ACK.
> UDP a 120-123: el **123 (ntp)** no respondió (abierto); 120/121/122 devolvieron **ICMP port unreachable** (cerrados).

### Port scanning con Nmap
El escáner de facto. Ver nota: [[Nmap]].

- Scan por defecto = **1000 puertos** más comunes (`/usr/share/nmap/nmap-services`). Genera ~72 KB; un `-p-` completo genera ~4 MB pero encuentra más.
- **`-sS`** (SYN / "stealth", por defecto con root): no completa el handshake, rápido, no llega a la capa de aplicación.
- **`-sT`** (connect): usa la API de sockets, sin privilegios, más lento; fallback sin raw sockets.
- **`-sU`** (UDP), combinable con `-sS`.
- **`-sn`** (ping sweep): no solo ICMP echo; también SYN a 443, ACK a 80 e ICMP timestamp.
- **`-O` / `--osscan-guess`** (OS fingerprinting, por TTL y TCP window; no 100% fiable).
- **`-sV`** (versiones) y **`-A`** (versión + scripts + OS + traceroute; mucho más tráfico).
- **`-oG`** (greppable) para filtrar hosts vivos.
- **NSE** (`--script`): scripts en `/usr/share/nmap/scripts` para DNS, brute, vulns, etc.
- Windows/LOLBAS: `Test-NetConnection -Port` y one-liner con `Net.Sockets.TcpClient`.

### SMB enumeration
Protocolo con largo historial de fallos. **SMB = TCP/445**; **NetBIOS = TCP/139** (protocolos distintos, se suelen enumerar juntos por compatibilidad NBT). Ver nota: [[SMB Enumeration]].

- Nmap `-p 139,445` para descubrir; **nbtscan** para nombres NetBIOS (descriptivos del rol del host).
- NSE `smb-os-discovery`, `smb-enum-*` (requieren SMBv1 en muchos casos).
- Windows/LOLBAS: `net view \\host /all` (el `/all` muestra shares administrativos con `$`).

### SMTP enumeration
Servidores de correo mal configurados permiten **enumerar usuarios** con `VRFY` (verifica una dirección) y `EXPN` (miembros de una lista). Ver nota: [[SMTP User Enumeration]].

- Respuestas distintas revelan usuarios válidos: `252` (aceptado) vs `550` (usuario desconocido).
- Automatizable con `nc`, script Python o, desde Windows, con el cliente **Telnet** (`dism /online /Enable-Feature /FeatureName:TelnetClient`, requiere admin).

### SNMP enumeration
**UDP/161**, stateless → susceptible a spoofing/replay. SNMP v1/v2/v2c **sin cifrado** y con community strings por defecto (`public` lectura, `private` escritura). Filtra muchísima info del host. Ver nota: [[SNMP Enumeration]].

- **MIB tree**: base de datos en árbol; cada hoja (OID) es una variable consultable.
- Descubrir: `nmap -sU --open -p 161` o brute de community strings con **onesixtyone**.
- Consultar: **snmpwalk** (con community + versión). OIDs útiles Windows: usuarios (`1.3.6.1.4.1.77.1.2.25`), procesos (`1.3.6.1.2.1.25.4.2.1.2`), software (`1.3.6.1.2.1.25.6.3.1.2`), puertos TCP (`1.3.6.1.2.1.6.13.1.3`).

> [!warning] Errores comunes / cosas que fallan
> - Olvidar **UDP** por centrarse en TCP: hay vectores tras 53, 123, 161...
> - Fiarse ciegamente de OS fingerprinting y banners: son *aproximados* y los banners pueden ser falseados.
> - No repetir la enumeración sobre los **nuevos hosts** descubiertos (el proceso es cíclico).
> - En UDP, tomar "no respuesta" como "abierto" cuando puede ser un firewall dropeando ICMP.

---

## Ejemplo práctico
> [!example] Flujo típico contra un dominio
> 1. `whois dominio.com` → contactos, name servers, rango IP (reverse).
> 2. Google dorks + Shodan + repos → subdominios, tecnologías, credenciales filtradas.
> 3. `host -t ns/mx/txt`, luego `dnsenum` / `dnsrecon -t axfr` para intentar *zone transfer*.
> 4. `nmap -sn` sweep → hosts vivos; luego `-p- -sV` sobre los interesantes.
> 5. Por servicio: SMB (`nbtscan`, NSE), SMTP (`VRFY`), SNMP (`snmpwalk`).
> 6. Repetir sobre cada host/subdominio nuevo.

## Relacionado
- Técnicas: [[Google Dorks]] · [[DNS Enumeration]] · [[Nmap]] · [[SMB Enumeration]] · [[SMTP User Enumeration]] · [[SNMP Enumeration]]
- Cheatsheet: [[Enumeracion]]
- Máquinas donde lo usé: [[]]

## Dudas
> [!question]
> - ¿Cuándo compensa `-sT` sobre `-sS` además de "sin privilegios" y proxies?
> - Práctica pendiente: intentar un *zone transfer* real y ver qué se filtra.
