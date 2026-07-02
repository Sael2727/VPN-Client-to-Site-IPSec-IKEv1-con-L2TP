# 🔐 VPN Client-to-Site — IPSec IKEv1 + L2TP

<div align="center">

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![Linux](https://img.shields.io/badge/Cliente-Linux-yellow?style=for-the-badge&logo=linux)
![IPSec](https://img.shields.io/badge/IPSec-IKEv1-green?style=for-the-badge)
![L2TP](https://img.shields.io/badge/Tunnel-L2TP%2FPPP-purple?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-orange?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-red?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Implementación y verificación de una **VPN Client-to-Site punto a multipunto usando L2TP protegido con IPSec IKEv1**. Un cliente Ubuntu Linux actúa como usuario remoto, el router **R1-SERVER** funciona como servidor VPN y la **VPC-LAN** representa la red interna a la que se accede de forma segura.

El cliente establece primero la seguridad **IPSec con IKEv1** y luego levanta la sesión **L2TP/PPP**. Al completarse la conexión, el router asigna al cliente una dirección del pool VPN y se crea la interfaz `ppp0` en Linux, desde donde puede acceder a la red interna `10.7.25.64/27`.

> 💡 **Clave técnica:** A diferencia de las VPNs Site-to-Site, aquí no existe interfaz de túnel en el router. Las sesiones L2TP se crean dinámicamente usando `Virtual-Template` + `vpdn-group`. IPSec protege el tráfico **UDP/1701** (L2TP) en **modo transport**, no tunnel. En el cliente Linux se usan **StrongSwan** para IPSec y **XL2TPD/PPP** para L2TP.

---

## 🗺️ Topología de Red

Un cliente Linux remoto se conecta al servidor VPN (R1-SERVER) a través de un ISP simulado. El router asigna una IP del pool VPN al cliente y le da acceso a la LAN interna donde está la VPC-LAN.

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara | Función |
|:-----------:|:--------:|:-------------:|---------|
| ISP | Ethernet0/0 | 10.7.25.2/30 | Conexión hacia R1-SERVER |
| ISP | Ethernet0/1 | 10.7.25.6/30 | Conexión hacia Cliente-Linux |
| R1-SERVER | Ethernet0/0 | 10.7.25.1/30 | WAN / Servidor VPN |
| R1-SERVER | Ethernet0/1 | 10.7.25.65/27 | Gateway LAN interna |
| Cliente-Linux | ens4 | 10.7.25.5/30 | Interfaz conectada al ISP |
| **Cliente-Linux** | **ppp0** | **10.7.25.193/32** | **IP recibida por L2TP** |
| VPC-LAN | eth0 | 10.7.25.66/27 | Host interno |
| **Pool VPN** | L2TP-POOL | **10.7.25.193 – 10.7.25.206** | **Direcciones para clientes VPN** |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Tipo de VPN | Client-to-Site punto a multipunto |
| Protocolo de túnel | L2TP / PPP |
| Protección | IPSec IKEv1 modo transport |
| Cifrado IKEv1 | AES-256, SHA, Grupo 5 |
| Clave precompartida (PSK) | VPN12345 |
| Transform-set IPSec | TS-L2TP: esp-aes 256 esp-sha-hmac (transport) |
| Tráfico protegido | UDP/1701 (L2TP) |
| Autenticación PPP | MS-CHAP-v2 |
| Usuario L2TP | vpnuser |
| Contraseña L2TP | vpnpass |
| Pool de IPs VPN | 10.7.25.193 – 10.7.25.206 |
| Red interna accesible | 10.7.25.64/27 |
| Herramientas Linux | StrongSwan + XL2TPD + PPP |

---

## 🔍 Funcionamiento

El proceso de conexión sigue estas etapas:

**1. Conectividad inicial** — El cliente Linux tiene la interfaz `ens4` con `10.7.25.5/30` y una ruta hacia el servidor VPN `10.7.25.1` vía ISP `10.7.25.6`.

**2. Negociación IPSec IKEv1** — StrongSwan en el cliente negocia IKEv1 con R1-SERVER usando PSK `VPN12345`. IPSec se levanta en modo transport protegiendo únicamente el tráfico **UDP/1701**.

**3. Sesión L2TP/PPP** — XL2TPD establece el túnel L2TP sobre el canal IPSec. PPP autentica al usuario con MS-CHAP-v2 usando `vpnuser/vpnpass`.

**4. Asignación de IP** — R1-SERVER asigna al cliente una dirección del pool `L2TP-POOL`. Se crea la interfaz `ppp0` en Linux con la IP `10.7.25.193`.

**5. Acceso a la LAN** — Con la ruta `10.7.25.64/27 dev ppp0`, el cliente alcanza la VPC-LAN interna con ping y tracepath.

---

## 🚀 Scripts de Configuración

### ISP
```cisco
hostname ISP
ip cef
interface Ethernet0/0
 description WAN_HACIA_R1_SERVER
 ip address 10.7.25.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description WAN_HACIA_CLIENTE_LINUX
 ip address 10.7.25.6 255.255.255.252
 no shutdown
```

### R1-SERVER — Configuración Principal
```cisco
hostname R1-SERVER
ip cef
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_INTERNA
 ip address 10.7.25.65 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.2

aaa new-model
aaa authentication ppp L2TP-AUTH local
aaa authorization network L2TP-AUTH local
username vpnuser password 0 vpnpass
ip local pool L2TP-POOL 10.7.25.193 10.7.25.206

vpdn enable
vpdn-group L2TP-GROUP
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication

interface Virtual-Template1
 description CLIENTES_L2TP
 ip unnumbered Ethernet0/1
 peer default ip address pool L2TP-POOL
 ppp authentication ms-chap-v2 L2TP-AUTH
 ppp authorization L2TP-AUTH
 ppp ipcp dns 8.8.8.8 1.1.1.1
 no shutdown

crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
crypto isakmp key VPN12345 address 0.0.0.0 0.0.0.0
crypto isakmp keepalive 10 3

crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac
 mode transport
crypto dynamic-map DYN-L2TP 10
 set transform-set TS-L2TP
crypto map VPN-MAP 10 ipsec-isakmp dynamic DYN-L2TP
interface Ethernet0/0
 crypto map VPN-MAP
```

### Cliente Linux — Configuración de Red
```bash
sudo ip link set ens4 up
sudo ip addr flush dev ens4
sudo ip addr add 10.7.25.5/30 dev ens4
sudo ip route add 10.7.25.1/32 via 10.7.25.6 dev ens4
sudo apt update
sudo apt install strongswan xl2tpd ppp -y
```

### `/etc/ipsec.conf`
```
config setup
    uniqueids=no

conn L2TP-PSK
    keyexchange=ikev1
    authby=secret
    type=transport
    left=10.7.25.5
    leftprotoport=17/1701
    right=10.7.25.1
    rightprotoport=17/1701
    ike=aes256-sha1-modp1536!
    esp=aes256-sha1!
    auto=add
```

### `/etc/ipsec.secrets`
```
10.7.25.5 10.7.25.1 : PSK "VPN12345"
```

### `/etc/xl2tpd/xl2tpd.conf`
```
[lac cisco-l2tp]
lns = 10.7.25.1
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes
```

### `/etc/ppp/options.l2tpd.client`
```
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-mschap-v2
noccp
noauth
mtu 1400
mru 1400
name vpnuser
password vpnpass
usepeerdns
```

### Comandos para levantar la VPN
```bash
sudo systemctl restart strongswan-starter
sudo systemctl restart xl2tpd
sudo ipsec up L2TP-PSK
echo "c cisco-l2tp" | sudo tee /var/run/xl2tpd/l2tp-control
ip addr show ppp0
sudo ip route add 10.7.25.64/27 dev ppp0
ping 10.7.25.65
ping 10.7.25.66
tracepath 10.7.25.66
```

---

## ✅ Verificación

```cisco
show ip interface brief
show running-config | section vpdn
show running-config interface virtual-template1
show ip local pool L2TP-POOL
show running-config | section crypto
show vpdn session
show users
show crypto isakmp sa
show crypto ipsec sa
show crypto session
```

| Comando / Acción | Estado esperado |
|:----------------:|-----------------|
| `show ip interface brief` en R1-SERVER | Virtual-Access activa |
| `ip addr show ppp0` en Linux | IP `10.7.25.193` asignada |
| `show vpdn session` | 1 túnel y 1 sesión L2TP activa, usuario `vpnuser` |
| `show users` | `vpnuser` conectado con peer `10.7.25.193` |
| `show crypto isakmp sa` | QM_IDLE |
| `show crypto ipsec sa` | encaps/decaps activos (UDP/1701) |
| `show crypto session` | UP-ACTIVE con peer `10.7.25.5` |
| `ping 10.7.25.65` desde Linux | Exitoso (gateway LAN) |
| `ping 10.7.25.66` desde Linux | Exitoso (VPC-LAN) |
| `tracepath 10.7.25.66` desde Linux | 2 saltos hasta la VPC |

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Interfaces ISP y R1-SERVER | ✅ up/up |
| Configuración VPDN / Virtual-Template1 | ✅ Correcta |
| Pool L2TP-POOL configurado | ✅ 10.7.25.193 – 10.7.25.206 |
| Configuración crypto IPSec/IKEv1 en R1-SERVER | ✅ Dynamic-map aplicado en WAN |
| Conectividad previa del cliente hacia R1-SERVER | ✅ Ping exitoso |
| Archivos ipsec.conf, ipsec.secrets, xl2tpd.conf, options.l2tpd.client | ✅ Configurados |
| IPSec IKEv1 levantado (`ipsec up L2TP-PSK`) | ✅ Conexión establecida |
| Interfaz ppp0 creada con IP del pool | ✅ 10.7.25.193/32 |
| Sesión L2TP activa en R1-SERVER | ✅ `show vpdn session` |
| Usuario vpnuser conectado | ✅ `show users` |
| IKEv1 SA | ✅ QM_IDLE |
| IPSec SA | ✅ encaps/decaps activos |
| Sesión crypto | ✅ UP-ACTIVE |
| Ping / Tracepath hacia LAN interna | ✅ Exitoso en 2 saltos |

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`SaelGerman_2025-0725_Script_L2TP_IPSec_IKEv1_Client_to_Site.txt`](SaelGerman_2025-0725_Script_L2TP_IPSec_IKEv1_Client_to_Site.txt) | Scripts de configuración Cisco IOS y Linux |
| [`SaelGerman_2025-0725_L2TP_IPSec_IKEV1_Client_to_Site_P2.pdf`](SaelGerman_2025-0725_L2TP_IPSec_IKEV1_Client_to_Site_P2.pdf) | Documentación técnica completa |

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología general VPN Client-to-Site L2TP + IPSec IKEv1](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/01_topologia_l2tp_ipsec_ikev1_client_to_site.png)
- 📸 [Figura 2 — Archivo xl2tpd.conf limpio en Cliente-Linux](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/02_cliente_linux_xl2tpd_conf_limpio.png)
- 📸 [Figura 3 — Interfaces del ISP](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/03_isp_show_ip_interface_brief.png)
- 📸 [Figura 4 — Interfaces de R1-SERVER (incluye Virtual-Access)](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/04_r1_server_show_ip_interface_brief.png)
- 📸 [Figura 5 — Configuración VPDN/L2TP en R1-SERVER](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/05_r1_server_show_running_config_seccion_vpdn.png)
- 📸 [Figura 6 — Configuración de Virtual-Template1](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/06_r1_server_show_running_config_virtual_template1.png)
- 📸 [Figura 7 — Pool L2TP-POOL de direcciones VPN](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/07_r1_server_ip_local_pool_l2tp.png)
- 📸 [Figura 8 — Configuración crypto IPSec/IKEv1 en R1-SERVER](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/08_r1_server_show_running_config_seccion_crypto.png)
- 📸 [Figura 9 — Interfaces ens4 y ppp0 del Cliente-Linux](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/09_cliente_linux_ip_a_ens4_y_ppp0.png)
- 📸 [Figura 10 — Tabla de rutas del Cliente-Linux](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/10_cliente_linux_ip_route.png)
- 📸 [Figura 11 — Prueba de conectividad inicial hacia R1-SERVER](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/11_cliente_linux_ping_servidor_vpn.png)
- 📸 [Figura 12 — Archivo /etc/ipsec.conf del Cliente-Linux](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/12_cliente_linux_ipsec_conf.png)
- 📸 [Figura 13 — Archivo /etc/ipsec.secrets del Cliente-Linux](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/13_cliente_linux_ipsec_secrets.png)
- 📸 [Figura 14 — Archivo /etc/xl2tpd/xl2tpd.conf del Cliente-Linux](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/14_cliente_linux_xl2tpd_conf.png)
- 📸 [Figura 15 — Archivo /etc/ppp/options.l2tpd.client](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/15_cliente_linux_ppp_options_l2tpd_client.png)
- 📸 [Figura 16 — Levantamiento de IPSec (ipsec up L2TP-PSK)](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/16_cliente_linux_ipsec_up_l2tp_psk_success.png)
- 📸 [Figura 17 — Interfaz ppp0 creada con IP del pool](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/17_cliente_linux_ip_addr_show_ppp0.png)
- 📸 [Figura 18 — Sesión L2TP activa en R1-SERVER (show vpdn session)](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/18_r1_server_show_vpdn_session.png)
- 📸 [Figura 19 — Usuario vpnuser conectado (show users)](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/19_r1_server_show_users_vpnuser.png)
- 📸 [Figura 20 — Verificación IKEv1 en R1-SERVER (QM_IDLE)](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/20_r1_server_show_crypto_isakmp_sa_qm_idle.png)
- 📸 [Figura 21 — IPSec SA en R1-SERVER (encaps/decaps activos)](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/21_r1_server_show_crypto_ipsec_sa_encaps.png)
- 📸 [Figura 22 — Sesión crypto UP-ACTIVE en R1-SERVER](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/22_r1_server_show_crypto_session_up_active.png)
- 📸 [Figura 23 — Ping y tracepath hacia LAN interna desde Cliente-Linux](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/23_cliente_linux_ping_tracepath_lan_interna.png)
- 📸 [Figura 24 — IP de la VPC-LAN](SaelGerman_2025-0725_Capturas_L2TP_IPSec_IKEv1_Client_to_Site_GitHub/24_vpc_lan_show_ip.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_L2TP_IPSec_IKEV1_Client_to_Site_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/ADtxan91PLg)

---

## 📚 Referencias

1. Cisco Systems. *Configuring L2TP over IPSec for Remote Access VPN*. Documentación oficial Cisco IOS.
2. StrongSwan Project. *IKEv1 PSK Configuration with L2TP*. Documentación oficial StrongSwan.
3. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
