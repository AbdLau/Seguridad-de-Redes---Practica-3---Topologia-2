# Infraestructura 2 - VPN Remote-Site, Jump Server y Web Server Caja

## Video demostrativo

[Ver video en YouTube](ENLACE_DEL_VIDEO)

## Propósito del laboratorio

Diseñar una infraestructura con acceso remoto seguro: un cliente se conecta por VPN SSL a un Jump Server, que es el único punto autorizado para llegar a un Web Server de caja mediante HTTPS, RDP y SSH. Se publican aplicaciones con RemoteApp y RD Web Access, y se restringen los accesos de los usuarios sin privilegios (sin SSH a servidores) y con privilegios (Web, PuTTY y RDP en el Jump).

## Topología

```mermaid
graph TD
    ISP[Cloud1 - Gestión VMnet8] --> FG[FortiGate-1]
    R2[R2 - ISP] -->|port2 WAN 200.8.47.4/30| FG
    R1[R1 - Equipo de red] -->|200.8.47.0/30| R2
    R1 -->|f0/1 LAN usuarios| SW[Switch VLAN 10]
    SW --> U1[User1-1 sin privilegios]
    SW --> PU[PrivilegedUser con privilegios]
    FG -->|port3 10.8.47.0/29| JUMP[Jump Server 10.8.47.2]
    FG -->|port4 10.8.47.8/29| WEB[Web Server Caja 10.8.47.10]
    PU -.->|VPN SSL 200.8.47.6:10443| FG
```

![Topología](imagenes/01_topologia_gns3.png)

## Plan de direccionamiento

| Segmento | Red | Gateway | Equipo |
|---|---|---|---|
| Enlace R1 ↔ R2 | 200.8.47.0/30 | 200.8.47.1 (R2) | R1: 200.8.47.2 |
| Enlace R2 ↔ FortiGate | 200.8.47.4/30 | 200.8.47.5 (R2) | FortiGate port2: 200.8.47.6 |
| LAN usuarios VLAN 10 | 10.8.46.0/25 | 10.8.46.1 (R1) | User1-1 10.8.46.2, PrivilegedUser 10.8.46.5 (DHCP) |
| LAN Jump | 10.8.47.0/29 | 10.8.47.1 (FortiGate port3) | Jump 10.8.47.2 |
| LAN Web | 10.8.47.8/29 | 10.8.47.9 (FortiGate port4) | Web 10.8.47.10 |

## Configuración

- **R1 (equipo de red):** f0/0 hacia R2 (200.8.47.2/30), f0/1 hacia la LAN de usuarios (10.8.46.1/25), ruta por defecto hacia R2 y rutas estáticas a las LAN del Jump y del Web. DHCP para VLAN 10 con reserva para User1-1.
- **R2 (ISP):** enlaces hacia R1 y FortiGate, rutas hacia 10.8.46.0/25, 10.8.47.0/29 y 10.8.47.8/29.
- **Switch:** puertos de usuarios en VLAN 10 (access) y enlace a R1.
- **FortiGate:** port1 gestión (Cloud1), port2 WAN (200.8.47.6/30), port3 LAN Jump, port4 LAN Web. Rutas estáticas: 0.0.0.0/0 por port2, 200.8.47.0/30 y 10.8.46.0/25 por port2.
- **Jump Server (Windows Server 2019):** RD Session Host, RD Web Access y RD Connection Broker en grupo de trabajo con dominio lab.local. RemoteApp: PuTTY, RDP y Navegador en la colección "Coleccion". Restricción: PuTTY y RDP solo para `usuario_priv`.
- **Web Server Caja:** HTTPS (puerto 443) y SSH (puerto 22).

### Objetos y políticas

| Política | Origen | Destino | Servicio | Acción |
|---|---|---|---|---|
| VPN | usuario_priv (SSL-VPN) | JUMP_SERVER | HTTPS, RDP | ACCEPT |
| NO_PRIVILEGE_USER | VLAN10_USUARIOS | JUMP_SERVER | HTTPS | ACCEPT |
| EXPLICIT_DENY | VLAN10_USUARIOS | JUMP_SERVER | SSH | DENY |
| EXPLICIT_DENY_2 | VLAN10_USUARIOS | WEB_CAJA | SSH | DENY |
| JUMP_TO_WEB | JUMP_SERVER | WEB_CAJA | HTTP, HTTPS, RDP, SSH | ACCEPT |
| DENY_JUMP_TO_WEB | JUMP_SERVER | WEB_CAJA | ALL | DENY |

## Pruebas y resultados

### Usuario sin privilegios
Solo HTTPS al Jump; SSH bloqueado hacia Jump y Web.

![User1-1 curl y SSH](imagenes/08_user_sinpriv_curl_ssh.png)

![Forward traffic usuario sin privilegios](imagenes/22_log_forward_user_sinpriv.png)

### Usuario con privilegios
Acceso directo RDP bloqueado; acceso por VPN al portal con Web, PuTTY y RDP.

![RDP directo bloqueado](imagenes/09_privilegeduser_rdp_directo.png)

![Login VPN](imagenes/10_vpn_login.png)

![Portal RDWeb por VPN](imagenes/12_vpn_remoteapp_apps.png)

![Log VPN](imagenes/21_log_vpn_events.png)

### Jump hacia Web Server
Permitidos HTTPS, SSH y RDP; denegado el resto y el ping.

![Pruebas desde el Jump](imagenes/13_jump_test_web_443_22_3389.png)

![Pruebas denegadas desde el Jump](imagenes/14_jump_test_web_8080_ping.png)

![Log JUMP_TO_WEB y DENY](imagenes/23_log_forward_jump_web.png)

### Políticas

![Lista de políticas](imagenes/15_politicas_lista.png)

![Política VPN](imagenes/20_pol_vpn.png)

![NO_PRIVILEGE_USER](imagenes/18_pol_no_privilege_user.png)

![EXPLICIT_DENY](imagenes/19_pol_explicit_deny.png)

![JUMP_TO_WEB](imagenes/16_pol_jump_to_web.png)

![DENY_JUMP_TO_WEB](imagenes/17_pol_deny_jump_to_web.png)

## Scripts

Configuración de IP de los equipos finales en [`scripts/ip_equipos_infra2.sh`](scripts/ip_equipos_infra2.sh).

## Running-configs

Pendientes: `configs/r1_running.txt`, `configs/r2_running.txt` y `configs/fortigate_backup.conf`.

