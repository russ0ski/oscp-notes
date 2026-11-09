---
tipo: tecnica
fase: pivoting
so: linux
tags: [oscp, tecnica, pivoting, linux]
---

# Proxychains

> [!abstract] Qué es y cuándo usarla
> Fuerza el tráfico de herramientas de terceros a través de un proxy **SOCKS/HTTP** (o una cadena de ellos). Imprescindible cuando usas un **dynamic / remote dynamic port forward** (SOCKS) con herramientas que no hablan SOCKS de forma nativa (`smbclient`, `nmap`, `psql`...).

## Requisitos previos
- Un puerto **SOCKS** ya abierto (p.ej. por [[SSH Port Forwarding]] con `-D` o `-R <port>`).
- La herramienta debe ser un **binario dinámico** (usa `LD_PRELOAD` para enganchar libc) → **no** funciona en binarios estáticos.

## Pasos
```bash
# 1) Configurar el proxy en /etc/proxychains4.conf (sección [ProxyList], al final)
#    socks5 <IP> <PORT>   (socks4 también vale; socks5 soporta auth, IPv6, UDP/DNS)
tail /etc/proxychains4.conf
# [ProxyList]
# socks5 127.0.0.1 9998

# 2) Anteponer proxychains al comando
proxychains smbclient -L //172.16.50.217/ -U hr_admin --password=Welcome1234
sudo proxychains nmap -vvv -sT --top-ports=20 -Pn -n 10.4.50.64   # nmap: SIEMPRE -sT (connect) -Pn
proxychains psql -h 10.4.50.215 -U postgres
```

> [!tip] Acelerar escaneos: los timeouts por defecto son altísimos. Baja `tcp_read_time_out` y `tcp_connect_time_out` en `proxychains4.conf` para que descarte conexiones muertas más rápido.

> [!warning] Detección / problemas
> - `nmap` por SOCKS solo con **`-sT`** (connect scan) y **`-Pn`**; los scans SYN/raw no pasan por el proxy.
> - No sirve para binarios **estáticos** ni para operaciones de red exóticas.
> - La opción nativa `nmap --proxies` está "under development" → usa proxychains.

## Visto en
- Módulo: [[19 - Port Redirection and SSH Tunneling]]
- Máquinas: [[]]
