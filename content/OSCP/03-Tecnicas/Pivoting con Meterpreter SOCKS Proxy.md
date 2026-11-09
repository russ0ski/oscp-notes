---
tipo: tecnica
fase: pivoting
so: linux
tags: [oscp, tecnica, pivoting, metasploit]
fecha: 2026-10-09
---

# Pivoting con Meterpreter SOCKS Proxy

> [!abstract] Qué es y cuándo usarla
> Cuando una sesión Meterpreter tiene visibilidad de una red interna no accesible directamente desde Kali, se puede levantar un proxy SOCKS dentro de Metasploit y enrutar el tráfico de esa red a través de la sesión comprometida. Combinado con un túnel SSH local, permite usar el navegador (o `proxychains`) de la Kali personal como si estuviera dentro de la red objetivo.

## Requisitos previos
- Sesión Meterpreter activa con acceso a la red interna objetivo.
- Metasploit corriendo en una instancia Kali con conectividad hacia la Kali personal (ej. Kali en la nube + SSH).

## Pasos
```
# Dentro de Metasploit
use auxiliary/server/socks_proxy
run -j                                  # arranca el proxy SOCKS en background (localhost:1080 por defecto)

sessions                                # obtener el ID de la sesión meterpreter activa
route add <red_objetivo> <mascara> <id_sesion>    # ej. route add 172.30.0.1 255.255.0.0 2
```
```bash
# Desde la Kali personal: túnel SSH local hacia la Kali donde corre el proxy SOCKS
ssh -fN -L localhost:1080:localhost:1080 <usuario>@<IP_kali_con_proxy>
```
Configurar el proxy SOCKS v5 (`127.0.0.1:1080`) en el navegador (Firefox: Settings → Network Settings → Manual proxy) o en `proxychains.conf` para que el tráfico pase por el túnel.

> [!warning] Detección / problemas
> - `exit` en una shell activa de Meterpreter puede cerrar la sesión por error — usar `background` en vez de `exit` para volver al prompt de Metasploit sin perder la sesión.
> - El `route add` debe apuntar al ID de sesión correcto — si la sesión muere y se reabre con otro ID, hay que repetir el `route add`.
> - Escanear rangos grandes (`/16`) a través del proxy es lento — reducir a `/24` cuando sea posible.

## Visto en
- Módulo: [[26 - Attacking AWS Cloud Infrastructure]]
- Relacionado: [[Metasploit]]
- Máquinas: [[]]
