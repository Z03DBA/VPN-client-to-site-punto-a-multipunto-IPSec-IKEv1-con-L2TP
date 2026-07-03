
# 🔒 Implementación de VPN Client-to-Site (L2TP/IPsec) en Entorno Cisco e Integración con Kali Linux

Este repositorio contiene la documentación técnica, los scripts de aprovisionamiento y las directivas de configuración utilizadas para el despliegue exitoso de una infraestructura de acceso remoto seguro **Client-to-Site**. El proyecto simula un escenario corporativo real utilizando el protocolo **L2TP** (Layer 2 Tunneling Protocol) encapsulado y protegido bajo el estándar criptográfico **IPsec** en modo transporte, estableciendo un túnel seguro desde una estación de auditoría **Kali Linux** hacia un Gateway perimetral **Cisco IOS**.

---

## 🗺️ Arquitectura de la Topología y Direccionamiento

La red está diseñada bajo un modelo de aislamiento perimetral estricto, dividiéndose en tres segmentos principales:

*   **Segmento Público (Simulación de Internet):** Red `192.8.39.0/24` donde interactúan la interfaz externa del Gateway corporativo y el endpoint del trabajador remoto.
*   **Segmento Privado (LAN Corporativa):** Red `172.16.10.0/24` donde residen los activos internos y servidores de producción a proteger.
*   **Segmento Virtual (VPN Pool):** Rango dinámico `192.168.100.50` al `192.168.100.100` asignado exclusivamente a los clientes autenticados de forma remota.

### 📋 Matriz de Direccionamiento

| Dispositivo | Interfaz Física / Virtual | Dirección IP | Máscara de Subred | Puerta de Enlace (GW) |
| :--- | :--- | :--- | :--- | :--- |
| **Router R1 (Servidor)** | GigabitEthernet1/0 (WAN) | `192.8.39.1` | `255.255.255.0` | *N/A* |
| **Router R1 (Servidor)** | GigabitEthernet2/0 (LAN) | `172.16.10.1` | `255.255.255.0` | *N/A* |
| **Kali Linux (Cliente)** | eth0 (Física) | `192.8.39.11` | `255.255.255.0` | `192.8.39.1` |
| **Kali Linux (Cliente)** | ppp0 (Túnel Virtual) | *Dinámica (Pool)* | `255.255.255.255`| *Asignada por R1* |
| **PC1 (Host Interno)** | VPCS Node | `172.16.10.100` | `255.255.255.0` | `172.16.10.1` |

---

## 🛠️ Especificaciones Criptográficas y de Seguridad

Para garantizar la confidencialidad, integridad y autenticidad del tráfico de extremo a extremo, se implementó una suite de seguridad robusta:

### IPsec Fase 1 (ISAKMP Policy)
*   **Algoritmo de Cifrado:** AES-256 (Advanced Encryption Standard).
*   **Algoritmo de Hash:** SHA-256 (Secure Hash Algorithm).
*   **Método de Autenticación:** Pre-Shared Key (Clave precompartida: `cisco123`).
*   **Grupo Diffie-Hellman:** Group 14 (Módulo de 2048 bits para prevenir ataques de fuerza bruta).

### IPsec Fase 2 (Transform-Set)
*   **Encapsulación:** ESP (Encapsulating Security Payload) con `esp-aes 256` y `esp-sha256-hmac`.
*   **Modo de Operación:** **Modo Transporte** (`mode transport`). *Nota de ingeniería: Se utiliza modo transporte para optimizar la sobrecarga de la cabecera IP, delegando la encapsulación multipunto al protocolo L2TP superior.*

### Capa de Autenticación de Usuario (PPP)
*   **Mecanismos:** CHAP (Challenge Handshake Authentication Protocol) y PAP (Password Authentication Protocol).
*   **Base de Datos:** Local (RBAC integrado en el Gateway).

## 🚀 Despliegue de Configuraciones

### 1. Configuración del Servidor Perimetral (Cisco IOS)
```text
! Habilitación de Virtual Private Dialup Network
vpdn enable
!
! Configuración de Fase 1 ISAKMP
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key cisco123 address 0.0.0.0 0.0.0.0
!
! Configuración de Fase 2 Transform-Set
crypto ipsec transform-set L2TP_SET esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto dynamic-map DYN_L2TP_MAP 10
 set transform-set L2TP_SET
!
crypto map L2TP_MAP 10 ipsec-isakmp dynamic DYN_L2TP_MAP
!
! Aplicación de directivas criptográficas a la interfaz pública
interface GigabitEthernet1/0
 ip address 192.8.39.1 255.255.255.0
 crypto map L2TP_MAP
 no shutdown
!
interface GigabitEthernet2/0
 ip address 172.16.10.1 255.255.255.0
 no shutdown
!
! Aprovisionamiento del Pool IP y Credenciales de Acceso
ip local pool L2TP_POOL 192.168.100.50 192.168.100.100
username zoe password vpnpassword123
!
! Definición del Grupo L2TP y enlace a la plantilla virtual
vpdn-group L2TP_SERVER_SERVER
 accept-dialin
  protocol l2tp
  virtual-template 1
 terminate-from local
!
interface Virtual-Template1
 ip unnumbered GigabitEthernet2/0
 peer default ip address pool L2TP_POOL
 ppp authentication chap pap

```

### 2. Aprovisionamiento del Cliente Endpoint (Kali Linux)

El cliente remoto utiliza **StrongSwan** para la orquestación criptográfica de IPsec y **xl2tpd** como el demonio L2TP de datos. Las directivas de inyección en archivos se ejecutan secuencialmente:

```bash
# Definición de la clave de intercambio secreta
echo ': PSK "cisco123"' | sudo tee /etc/ipsec.secrets

# Configuración estructural del archivo ipsec.conf
sudo tee /etc/ipsec.conf << 'EOF'
config setup
    charondebug="ike 2, knl 2, cfg 2"
conn cisco-l2tp
    authby=secret
    pfs=no
    auto=add
    keyexchange=ikev1
    type=transport
    left=%defaultroute
    leftprotoport=17/1701
    right=192.8.39.1
    rightprotoport=17/1701
    ike=aes256-sha256-modp2048!
    esp=aes256-sha256!
EOF

# Inicialización de servicios y establecimiento del túnel
sudo systemctl restart strongswan-starter xl2tpd
sudo ipsec rereadsecrets && sudo ipsec reload
sudo ipsec up cisco-l2tp
sudo sh -c 'echo "c cisco-vpn" > /var/run/xl2tpd/l2tp-control'

```

---

## 📊 Auditoría y Verificación del Entorno

Para validar la correcta integridad de los planos de control y de datos, se ejecutan las siguientes directivas de diagnóstico en el router **R1**:

### 1. Inspección del Plano de Cifrado (IPsec SAs)

Muestra las asociaciones criptográficas activas en memoria volátil:

```text
R1# show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
192.8.39.1      192.8.39.11     QM_IDLE           1001 ACTIVE

```

* **Análisis Técnico:** El estado `QM_IDLE` valida que la negociación IKEv1 concluyó con éxito y el canal seguro se mantiene en escucha activa de tráfico.

### 2. Verificación de Sesiones L2TP (VPDN Sessions)

Monitorea los túneles lógicos y la encapsulación de datos de capa 2:

```text
R1# show vpdn session
VPDN L2TP Session Information:
User Name: zoe, Session ID: 1, Tunnel ID: 1, State: estab

```

* **Análisis Técnico:** El estado `estab` certifica que el intercambio de credenciales vía PPP fue aceptado y el usuario `zoe` posee un túnel de datos dedicado.

### 3. Prueba de Conectividad End-to-End (ICMP injection)

Inyección de paquetes ICMP desde el adaptador lógico `ppp0` de Kali Linux hacia el host interno protegido:

```bash
kali@auditoria:~# ping 172.16.10.100 -c 4
PING 172.16.10.100 (172.16.10.100) 56(84) bytes of data.
64 bytes from 172.16.10.100: icmp_seq=1 ttl=64 time=4.12 ms
64 bytes from 172.16.10.100: icmp_seq=2 ttl=64 time=3.89 ms

--- 172.16.10.100 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms

```

* **Conclusión:** La tasa de pérdida del 0% demuestra de manera contundente la correcta inyección, enrutamiento interno y desencapsulación transparente del tráfico cifrado.

