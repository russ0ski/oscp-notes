---
tags: [oscp, indice]
---

# OSCP — Índice

## Módulos
<!-- Claude Code añade aquí cada módulo -->
- [[06 - Information Gathering]] — OSINT pasivo + enumeración activa (DNS, SMB, SMTP, SNMP, Nmap)
- [[07 - Vulnerability Scanning]] — teoría (CVE/CVSS, tipos de scan), Nessus (auth/no-auth, plugins) y Nmap NSE (vuln, vulners)
- [[08 - Introduction to Web Application Attacks]] — metodología (black-box, OWASP), herramientas (Nmap, Wappalyzer, Gobuster, Burp), enumeración (headers, source, APIs REST) y XSS (stored/reflected/DOM) hasta privesc
- [[09 - Common Web Application Attacks]] — Directory Traversal, File Inclusion (LFI/RFI + PHP wrappers), File Upload y Command Injection → RCE/reverse shell
- [[10 - SQL Injection Attacks]] — teoría SQL (MySQL/MSSQL), explotación manual (error-based, UNION, blind boolean/time-based) y RCE (xp_cmdshell, INTO OUTFILE) + automatización con sqlmap
- [[11 - Phishing Basics]] — ingeniería social (pretexto, palancas, smishing/vishing, Gen AI/deepfakes), defensas a sortear (filtros, MotW, macros, password managers, MFA) y lab de credential phishing clonando Zoom
- [[12 - Client-side Attacks]] — recon del target (metadatos `exiftool`, fingerprinting con Canarytokens), macros VBA en Office (MotW/Protected View, PowerCat) y Windows library files (`.Library-ms` + WebDAV + `.lnk`) para foothold
- [[13 - Locating Public Exploits]] — encontrar y evaluar exploits públicos (Exploit-DB, Packet Storm, GitHub, Google dorks), recursos offline (`searchsploit`, frameworks, NSE) y flujo completo versión→exploit→reverse shell (qdPM 9.1)
- [[14 - Fixing Exploits]] — adaptar exploits públicos: memory corruption en C (teoría BOF, cross-compilar con `mingw-w64`, socket/return address/shellcode, off-by-one) y web exploits (HTTPS `verify=False`, credenciales, troubleshooting CSRF)
- [[15 - Antivirus Evasion]] — cómo funciona el AV (firmas/heurística/ML, engines, EDR/SIEM), bypass on-disk (packers/crypters) vs in-memory (PE injection, hollowing, hooking) y práctica: thread injection con PowerShell + `msfvenom psh-reflection` y automatización con Shellter
- [[16 - Password Attacks]] — ataques online a logins (Hydra: SSH/RDP spray/HTTP POST) y cracking offline (Hashcat/John, reglas, KeePass/SSH); hashes Windows: NTLM+Mimikatz→crack/Pass-the-Hash, Net-NTLMv2 con Responder→crack/relay (ntlmrelayx) y Credential Guard+memssp
- [[17 - Windows Privilege Escalation]] — situational awareness (users/grupos/SO/red/apps/procesos), info sensible en ficheros y artefactos de PowerShell (historial PSReadline, transcripts), winPEAS; abuso de servicios (Service Binary Hijacking, DLL Hijacking, Unquoted Service Paths), Scheduled Tasks, exploits de aplicación/kernel y Potato attacks (SeImpersonatePrivilege)
- [[18 - Linux Privilege Escalation]] — permisos y SUID/SGID, enumeración manual y automatizada (unix-privesc-check, LinPEAS); info confidencial expuesta (dotfiles, env, huellas de procesos/tcpdump); cron jobs y /etc/passwd escribibles; SUID/capabilities (GTFOBins), abuso de sudo y exploits de kernel
- [[19 - Port Redirection and SSH Tunneling]] — atravesar firewalls/segmentación desde un pivote: port forwarding con Socat, SSH tunneling (local -L, dynamic -D, remote -R, remote dynamic) + Proxychains y sshuttle, y herramientas Windows (ssh.exe, Plink, Netsh portproxy)
- [[20 - Tunneling Through Deep Packet Inspection]] — tunelizar cuando el DPI filtra por protocolo: HTTP tunneling con Chisel (reverse SOCKS proxy, glibc/ncat) y DNS tunneling con dnscat2 (exfil en subdominios, infil en TXT/CNAME/MX, port forward tipo ssh -L)
- [[21 - The Metasploit Framework]] — MSF de punta a punta: BBDD/workspaces/db_nmap, módulos auxiliary y exploit, payloads (staged vs non-staged, Meterpreter, msfvenom + multi/handler), post-explotación (getsystem/migrate, bypass UAC, Kiwi), pivoting (autoroute/socks_proxy/portfwd) y resource scripts
- [[22 - Active Directory Introduction and Enumeration]] — enumeración de AD (assumed breach): teoría (objetos/OUs/DC/grupos/LDAP/DN), manual con net.exe, LDAP+.NET/ADSI y PowerView (users/groups/computers, SPNs, Find-LocalAdminAccess, PsLoggedOn, ACLs/GenericAll, shares/GPP cpassword) y automática con SharpHound+BloodHound
- [[23 - Attacking Active Directory Authentication]] — NTLM vs Kerberos, credenciales cacheadas en LSASS (Mimikatz), password spraying (Spray-Passwords/crackmapexec/kerbrute) y ataques Kerberos: AS-REP Roasting (18200), Kerberoasting (13100), Silver Ticket (kerberos::golden) y DCSync (secretsdump)
- [[24 - Lateral Movement in Active Directory]] — moverse por el dominio: WMI/WinRM/WinRS, PsExec, DCOM, Pass-the-Hash, Overpass-the-Hash y Pass-the-Ticket; y persistencia: Golden Ticket (krbtgt) y Shadow Copy → NTDS.dit (secretsdump)
- [[25 - Enumerating AWS Cloud Infrastructure]] — descubrir infraestructura cloud (DNS/WHOIS, dnsenum, DevTools), enumerar buckets S3 sin credenciales (`cloud_enum`), recursos AWS CLI (AMIs/snapshots públicos, Account ID vía política S3) y, con credenciales comprometidas, enumeración IAM completa (`get-account-authorization-details`, JMESPath, Pacu) hasta extraer rutas de escalado de privilegios
- [[26 - Attacking AWS Cloud Infrastructure]] — ataques a CI/CD sobre AWS: Poisoned Pipeline Execution editando un `Jenkinsfile` tras encontrar secretos en el historial de Git de un bucket S3 mal configurado, creación de cuenta backdoor con credenciales de pipeline; y dependency confusion publicando un paquete pip malicioso, pivoting con proxy SOCKS de Meterpreter, plugin de Jenkins vulnerable y robo de credenciales admin desde un Terraform state file
- [[27 - Assembling the Pieces]] — pentest completo (Challenge Lab Zero) contra BEYOND Finances: directory traversal no autenticado en WordPress (Duplicator, CVE-2020-11738) → robo y crackeo de clave SSH → privesc Linux (sudo git/GTFOBins, minado de historial de Git) → credential spraying (CrackMapExec) → phishing dirigido (Windows Library + WebDAV) → enumeración con BloodHound → pivoting (Metasploit + Chisel + Proxychains) → Kerberoasting → relay NTLM forzado desde un plugin de WordPress (ntlmrelayx) → credenciales cacheadas de un Domain Admin (Mimikatz) → Pass-the-Hash hasta el controlador de dominio

## Cheatsheets
- [[Enumeracion]] · [[Web]] · [[Explotacion y Shells]] · [[Transferencia de archivos]]
- [[Privesc Linux]] · [[Privesc Windows]] · [[Active Directory]]
- [[Pivoting y Tunneling]] · [[Password Attacks]] · [[Metasploit]] · [[AWS]] · [[CICD y Supply Chain]]

## Progreso (requiere plugin Dataview)
```dataview
TABLE estado, fecha FROM "01-Modulos" SORT file.name ASC
```

```dataview
TABLE plataforma, so, estado FROM "04-Maquinas" SORT fecha DESC
```
</content>
