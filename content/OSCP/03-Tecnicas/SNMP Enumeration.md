---
tipo: tecnica
tags: [oscp, enumeracion, windows]
fecha: 2026-09-26
---

# SNMP Enumeration

> [!abstract]
> SNMP (**UDP/161**) es stateless y, en v1/v2/v2c, sin cifrado y con *community strings* por defecto (`public` = lectura, `private` = escritura). Filtra usuarios, procesos, software y puertos del host.

## MIB tree
Base de datos en árbol; cada hoja es un **OID** consultable. OIDs útiles en Windows:
| OID | Contenido |
|---|---|
| `1.3.6.1.2.1.25.1.6.0` | Procesos del sistema (nº) |
| `1.3.6.1.2.1.25.4.2.1.2` | Programas en ejecución |
| `1.3.6.1.2.1.25.4.2.1.4` | Rutas de los procesos |
| `1.3.6.1.2.1.25.6.3.1.2` | Software instalado |
| `1.3.6.1.4.1.77.1.2.25` | Cuentas de usuario |
| `1.3.6.1.2.1.6.13.1.3` | Puertos TCP locales a la escucha |

## Descubrir SNMP
```bash
sudo nmap -sU --open -p 161 192.168.50.1-254 -oG open-snmp.txt

# Brute de community strings con onesixtyone
echo public > community; echo private >> community; echo manager >> community
for ip in $(seq 1 254); do echo 192.168.50.$ip; done > ips
onesixtyone -c community -i ips
```
| Flag | Qué hace |
|---|---|
| `-sU --open` | UDP, solo puertos abiertos |
| `-c` / `-i` (onesixtyone) | Fichero de communities / de IPs |

## Consultar con snmpwalk
```bash
snmpwalk -c public -v1 -t 10 192.168.50.151                          # todo el MIB
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.4.1.77.1.2.25          # usuarios
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.2.1.25.4.2.1.2         # procesos
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.2.1.25.6.3.1.2         # software
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.2.1.6.13.1.3          # puertos TCP
```
| Flag | Qué hace |
|---|---|
| `-c <string>` | Community string (normalmente `public`) |
| `-v1` | Versión SNMP |
| `-t 10` | Timeout 10s |

> [!tip]
> Cruza procesos + software para saber la versión exacta de un binario vulnerable. El OID de puertos TCP puede revelar servicios que solo escuchan en localhost (nuevos vectores tras un foothold).

## Relacionado
- Módulo: [[06 - Information Gathering]]
- Cheatsheet: [[Enumeracion]]
