# IPSec VPN — Basado en Enrutamiento (Route-Based / VTI)

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [¿Qué es una VPN Basada en Enrutamiento?](#21-qué-es-una-vpn-basada-en-enrutamiento)
   - [Virtual Tunnel Interface (VTI)](#22-virtual-tunnel-interface-vti)
   - [VPN Basada en Rutas vs. Basada en Políticas](#23-vpn-basada-en-rutas-vs-basada-en-políticas)
   - [Parámetros Criptográficos Utilizados](#24-parámetros-criptográficos-utilizados)
3. [Documentación de la Red](#3-documentación-de-la-red)
   - [Topología](#31-topología)
   - [Tabla de Dispositivos y Direccionamiento IP](#32-tabla-de-dispositivos-y-direccionamiento-ip)
4. [Scripts de Configuración](#4-scripts-de-configuración)
   - [Router R1 — Site A](#41-router-r1--site-a)
   - [Router R2 — Site B](#42-router-r2--site-b)
   - [Configuración de PCs (Hosts)](#43-configuración-de-pcs-hosts)
5. [Verificación del Túnel](#5-verificación-del-túnel)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Consideraciones de Seguridad](#7-consideraciones-de-seguridad)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo del Laboratorio

El objetivo de este laboratorio es **implementar y verificar una VPN Site-to-Site Point-to-Point basada en enrutamiento utilizando Virtual Tunnel Interfaces (VTI) con IPSec** sobre infraestructura Cisco IOS. A través de esta práctica se busca demostrar:

* La configuración de una interfaz de túnel lógica (`Tunnel0`) protegida con IPSec mediante el comando `tunnel protection ipsec profile`, eliminando la dependencia de Crypto Maps y ACLs de tráfico interesante del modelo *policy-based*.
* La protección criptográfica del tráfico de datos entre dos subredes LAN privadas (`20.25.37.128/25` y `20.25.37.0/25`) a través de un segmento de red pública simulado (`192.168.1.0/24`).
* La habilitación del **enrutamiento dinámico sobre el túnel** mediante OSPF, demostrando la principal ventaja del modelo *route-based* frente al *policy-based*.
* La validación del túnel VTI mediante comandos de verificación nativos de IOS y la comprobación de conectividad extremo a extremo con `ping` extendido entre hosts.

---

## 2. Marco Teórico

### 2.1 ¿Qué es una VPN Basada en Enrutamiento?

Una VPN basada en enrutamiento (*route-based*) es un modelo de implementación de IPSec en el que la decisión de cifrar el tráfico **no recae en una ACL** (como en el modelo *policy-based*), sino en la **tabla de enrutamiento**. El router envía tráfico al túnel de la misma manera que lo haría con cualquier otra interfaz: si la ruta del destino apunta a la interfaz de túnel, el paquete es cifrado y encapsulado automáticamente.

Esto se logra mediante una **Virtual Tunnel Interface (VTI)**, que es una interfaz lógica de Cisco IOS que actúa como el endpoint local del túnel IPSec. Todo el tráfico que entra a esta interfaz es cifrado por IPSec antes de ser enviado al peer remoto.

El modelo *route-based* es el **estándar moderno** para VPNs de infraestructura. Es la arquitectura sobre la que se construyen soluciones como DMVPN, FlexVPN y la mayoría de las implementaciones VPN en la nube (AWS VPN Gateway, Azure VPN Gateway, Google Cloud VPN).

### 2.2 Virtual Tunnel Interface (VTI)

La VTI es el componente central de este modelo. Desde el punto de vista del sistema operativo IOS, es simplemente una interfaz de red — igual que una interfaz Ethernet o una interfaz GRE convencional. La diferencia es que lleva asociado un **IPSec Profile**, que define qué parámetros criptográficos deben usarse para proteger el tráfico que la atraviesa.

Existen dos tipos de VTI:

| Tipo | Descripción | Uso |
|---|---|---|
| **Static VTI (sVTI)** | Tunnel con un peer remoto fijo y conocido de antemano. | VPNs site-to-site punto a punto. **Este laboratorio.** |
| **Dynamic VTI (dVTI)** | El router crea instancias VTI dinámicamente cuando recibe conexiones. | Concentradores VPN con múltiples clientes dinámicos (FlexVPN). |

El flujo de un paquete a través de una sVTI es:

```
Host → [R1 LAN] → Interfaz Tunnel0 → IPSec Profile (cifrado ESP) → Interfaz WAN → Internet → R2 WAN → IPSec (descifrado) → R2 LAN → Host destino
```

A diferencia del modelo *policy-based*, no existe una ACL que filtre qué tráfico entra al túnel — esa decisión la toma enteramente el enrutamiento.

### 2.3 VPN Basada en Rutas vs. Basada en Políticas

| Característica | Route-Based / VTI (este lab) | Policy-Based / Crypto Map |
|---|---|---|
| **Mecanismo de selección de tráfico** | Tabla de enrutamiento — la ruta apunta a `Tunnel0` | ACL de "tráfico interesante" en la Crypto Map |
| **Enrutamiento dinámico sobre el túnel** | ✅ Soportado — OSPF, EIGRP, BGP corren directamente sobre la VTI | ❌ No soportado directamente |
| **Configuración** | `tunnel protection ipsec profile` en la interfaz Tunnel | `crypto map` aplicado a la interfaz física WAN |
| **Escalabilidad** | Alta — una sola VTI puede ampliarse a DMVPN/FlexVPN | Baja — una Crypto Map entry por par de subredes |
| **Soporte multicast** | ✅ Nativo sobre la VTI | ❌ Requiere configuración adicional |
| **Número de SAs IPSec** | Una SA por túnel (todo el tráfico va por una sola SA) | Una SA por cada par de redes en la ACL |
| **Troubleshooting** | Más simple — el túnel es una interfaz visible con `show interface` | Más complejo — depende del estado de la ACL y la SA |
| **IOS soporte** | IOS 12.3(14)T+ | Todas las versiones IOS |
| **Compatibilidad inter-vendor** | Buena, pero depende de la implementación VTI del vendor | Universal — todos los vendors soportan Crypto Map |

La principal ventaja del modelo *route-based* es que **el túnel se comporta como un enlace de red estándar**, permitiendo correr protocolos de enrutamiento dinámico (OSPF, EIGRP, BGP) directamente sobre él — algo imposible en el modelo *policy-based* sin GRE adicional.

### 2.4 Parámetros Criptográficos Utilizados

| Parámetro | Valor configurado | Propósito |
|---|---|---|
| **Cifrado IKE (Fase 1)** | AES-256 | Cifrado simétrico del canal IKE |
| **Hash IKE (Fase 1)** | SHA-256 | Integridad de mensajes IKE |
| **Autenticación** | Pre-Shared Key (PSK) | Autenticación mutua de los peers |
| **Grupo Diffie-Hellman** | Grupo 14 (2048-bit MODP) | Intercambio seguro de clave simétrica |
| **Lifetime ISAKMP SA** | 86400 segundos (24 horas) | Duración del canal IKE antes de renegociar |
| **Cifrado ESP (Fase 2)** | AES-256 | Cifrado del tráfico de datos |
| **Autenticación ESP (Fase 2)** | SHA-256 HMAC | Integridad del tráfico de datos |
| **Modo IPSec** | Tunnel | Encapsulación completa del paquete IP original |
| **Lifetime IPSec SA** | 3600 segundos (1 hora) | Duración del túnel de datos antes de renegociar |
| **Enrutamiento sobre el túnel** | OSPF (Área 0) | Propagación dinámica de rutas LAN entre sitios |
| **Subnet del túnel VTI** | 10.25.37.0/30 | Enlace lógico punto a punto entre R1 y R2 |

---

## 3. Documentación de la Red

### 3.1 Topología

El laboratorio simula la interconexión segura de dos sitios corporativos a través de Internet. El segmento `192.168.1.0/24` representa la nube pública/ISP. Cada sitio tiene su propia subred LAN privada derivada de la matrícula `20250737`. La interfaz `Tunnel0` en cada router actúa como enlace lógico punto a punto entre los dos sitios, usando la subred `10.25.37.0/30`.

```
                              [ INTERNET / ISP ]
                              192.168.1.0/24
                              (Router ISP: 192.168.1.2)
                                    │
                   ┌────────────────┴────────────────┐
                   │ e0/0: 192.168.1.10              │ e0/0: 192.168.1.20
           ┌───────┴───────┐                 ┌───────┴───────┐
           │  Router R1    │◄═══ TÚNEL ══════►  Router R2    │
           │  (Site A)     │  IPSec VTI/VPN  │  (Site B)     │
           │  Tunnel0:     │  Route-Based    │  Tunnel0:     │
           │  10.25.37.1   │                 │  10.25.37.2   │
           └───────┬───────┘                 └───────┬───────┘
                   │ e0/1: 20.25.37.129/25           │ e0/1: 20.25.37.1/25
                   │                                 │
           ┌───────┴───────┐                 ┌───────┴───────┐
           │  Switch SW1   │                 │  Switch SW2   │
           └───┬───────┬───┘                 └───┬───────┬───┘
               │       │                         │       │
           ┌───┴──┐ ┌──┴───┐               ┌────┴─┐ ┌───┴──┐
           │ PC1  │ │ PC2  │               │ PC3  │ │ PC4  │
           │.130  │ │.131  │               │ .2   │ │ .3   │
           └──────┘ └──────┘               └──────┘ └──────┘
            20.25.37.128/25                 20.25.37.0/25
               SITE A                          SITE B

  ════════════════════════════════════════════════════════════════
  Flujo del túnel IPSec Route-Based (VTI):
    1. PC1 (20.25.37.130) hace ping a PC3 (20.25.37.2).
    2. R1 consulta su tabla de enrutamiento: la ruta hacia
       20.25.37.0/25 apunta a la interfaz Tunnel0 (OSPF).
    3. El paquete entra a Tunnel0 → IPSec Profile lo cifra (ESP).
    4. R1 encapsula el paquete cifrado y lo envía a 192.168.1.20.
    5. R2 recibe, descifra (IPSec) y desencapsula → entrega a PC3.
    6. No hay ACL de tráfico interesante — el enrutamiento decide.
  ════════════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Gateway | Rol |
|---|---|---|---|---|---|---|
| **ISP** | Router ISP | e0/0 | 192.168.1.2 | /24 | — | Enlace hacia R1 |
| | | e0/1 | 192.168.1.2 | /24 | — | Enlace hacia R2 |
| **R1** | Cisco IOS (Router) | e0/0 | 192.168.1.10 | /24 | 192.168.1.2 | Gateway WAN — Site A |
| | | e0/1 | 20.25.37.129 | /25 | — | Gateway LAN — Site A |
| | | Tunnel0 | 10.25.37.1 | /30 | — | VTI — Endpoint local del túnel |
| **R2** | Cisco IOS (Router) | e0/0 | 192.168.1.20 | /24 | 192.168.1.2 | Gateway WAN — Site B |
| | | e0/1 | 20.25.37.1 | /25 | — | Gateway LAN — Site B |
| | | Tunnel0 | 10.25.37.2 | /30 | — | VTI — Endpoint remoto del túnel |
| **SW1** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Site A |
| **SW2** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Site B |
| **PC1** | Host Linux / VPC | eth0 | 20.25.37.130 | /25 | 20.25.37.129 | Host Site A |
| **PC2** | Host Linux / VPC | eth0 | 20.25.37.131 | /25 | 20.25.37.129 | Host Site A |
| **PC3** | Host Linux / VPC | eth0 | 20.25.37.2 | /25 | 20.25.37.1 | Host Site B |
| **PC4** | Host Linux / VPC | eth0 | 20.25.37.3 | /25 | 20.25.37.1 | Host Site B |

---

## 4. Scripts de Configuración

### 4.1 Router R1 — Site A

```cisco
! ══════════════════════════════════════════════════════
! R1 — Site A | IPSec Route-Based VPN (VTI)
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R1

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.1.10 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteA
 ip address 20.25.37.129 255.255.255.128
 no shutdown

! ─── Ruta por defecto hacia Internet (ISP) ─────────────
ip route 0.0.0.0 0.0.0.0 192.168.1.2

! ─── PASO 1: ISAKMP Policy — IKE Fase 1 ───────────────
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! ─── Pre-Shared Key del peer remoto (R2) ───────────────
crypto isakmp key JordyITLA2026! address 192.168.1.20

! ─── PASO 2: Transform Set — IKE Fase 2 (ESP) ─────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ─── PASO 3: IPSec Profile ─────────────────────────────
! En route-based NO se usa Crypto Map en la interfaz WAN.
! En su lugar, el IPSec Profile se aplica directamente a Tunnel0.
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set security-association lifetime seconds 3600

! ─── PASO 4: Virtual Tunnel Interface (VTI) ────────────
interface Tunnel0
 description VPN-Route-Based-hacia-R2
 ip address 10.25.37.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel destination 192.168.1.20
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROFILE_VTI
 no shutdown

! ─── PASO 5: OSPF — Enrutamiento dinámico sobre el túnel
! Esta es la ventaja clave del modelo route-based:
! OSPF corre sobre Tunnel0 y aprende las rutas LAN remotas.
router ospf 1
 network 20.25.37.128 0.0.0.127 area 0    ! LAN local Site A
 network 10.25.37.0 0.0.0.3 area 0        ! Subred del túnel VTI
```

### 4.2 Router R2 — Site B

```cisco
! ══════════════════════════════════════════════════════
! R2 — Site B | IPSec Route-Based VPN (VTI)
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R2

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.1.20 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteB
 ip address 20.25.37.1 255.255.255.128
 no shutdown

! ─── Ruta por defecto hacia Internet (ISP) ─────────────
ip route 0.0.0.0 0.0.0.0 192.168.1.2

! ─── PASO 1: ISAKMP Policy — IKE Fase 1 ───────────────
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! ─── Pre-Shared Key del peer remoto (R1) ───────────────
crypto isakmp key JordyITLA2026! address 192.168.1.10

! ─── PASO 2: Transform Set — IKE Fase 2 (ESP) ─────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ─── PASO 3: IPSec Profile ─────────────────────────────
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set security-association lifetime seconds 3600

! ─── PASO 4: Virtual Tunnel Interface (VTI) ────────────
interface Tunnel0
 description VPN-Route-Based-hacia-R1
 ip address 10.25.37.2 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel destination 192.168.1.10
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROFILE_VTI
 no shutdown

! ─── PASO 5: OSPF — Enrutamiento dinámico sobre el túnel
router ospf 1
 network 20.25.37.0 0.0.0.127 area 0      ! LAN local Site B
 network 10.25.37.0 0.0.0.3 area 0        ! Subred del túnel VTI
```

### 4.3 Configuración de PCs (Hosts)

**PC1 y PC2 — Site A (`20.25.37.128/25`)**

```bash
# PC1
ip 20.25.37.130 255.255.255.128 20.25.37.129

# PC2
ip 20.25.37.131 255.255.255.128 20.25.37.129
```

**PC3 y PC4 — Site B (`20.25.37.0/25`)**

```bash
# PC3
ip 20.25.37.2 255.255.255.128 20.25.37.1

# PC4
ip 20.25.37.3 255.255.255.128 20.25.37.1
```

> **Nota:** Si los hosts son VPCs de PNETLab se usa el comando `ip` como se muestra. Si son máquinas Linux se configura con `ip addr add` e `ip route add default via`.

---

## 5. Verificación del Túnel

### 5.1 Verificar el estado de la interfaz Tunnel0

A diferencia del modelo *policy-based*, en el modelo *route-based* la interfaz de túnel es visible directamente con `show interface`:

```cisco
R1# show interface Tunnel0
```

*Salida esperada:*

```
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Description: VPN-Route-Based-hacia-R2
  Internet address is 10.25.37.1/30
  MTU 1400 bytes, BW 100 Kbit/sec, DLY 50000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation TUNNEL, loopback not set
  Tunnel source 192.168.1.10 (Ethernet0/0), destination 192.168.1.20
  Tunnel protocol/transport IPSEC/IP
  Tunnel protection via IPSec (profile "IPSEC_PROFILE_VTI")
```

> Si el estado es `down/down`, el problema generalmente es conectividad WAN entre los peers. Verificar primero `ping 192.168.1.20` desde R1.

---

### 5.2 Verificar el estado de IKE Fase 1 (ISAKMP SA)

```cisco
R1# show crypto isakmp sa
```

*Salida esperada:*

```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
192.168.1.20    192.168.1.10    QM_IDLE        1001    ACTIVE
```

> `QM_IDLE` confirma que la Fase 1 completó exitosamente. Si la interfaz Tunnel0 está `up/up` pero no hay ISAKMP SA, verificar que la PSK sea idéntica en ambos routers.

---

### 5.3 Verificar las IPSec SAs activas (Fase 2)

```cisco
R1# show crypto ipsec sa
```

*Salida esperada (fragmento):*

```
interface: Tunnel0
    Crypto map tag: Tunnel0-head-0, local addr 192.168.1.10

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   current_peer 192.168.1.20 port 500

    #pkts encaps: 18, #pkts encrypt: 18, #pkts digest: 18
    #pkts decaps: 18, #pkts decrypt: 18, #pkts verify: 18
```

> **Diferencia clave vs. policy-based:** Los identifiers `local/remote ident` muestran `0.0.0.0/0.0.0.0` — esto es correcto. La VTI protege **todo** el tráfico que entra a Tunnel0, no un subconjunto definido por ACL.

---

### 5.4 Verificar la tabla de enrutamiento OSPF

```cisco
R1# show ip route ospf
```

*Salida esperada:*

```
      20.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
O        20.25.37.0/25 [110/1001] via 10.25.37.2, 00:01:32, Tunnel0
```

> La ruta hacia la LAN del Site B (`20.25.37.0/25`) fue aprendida por OSPF **a través del túnel** (`via 10.25.37.2, Tunnel0`). Esta es la demostración de la ventaja del modelo *route-based*: enrutamiento dinámico nativo sobre IPSec.

---

### 5.5 Verificar conectividad extremo a extremo

Ejecutar desde **PC1** hacia **PC3**:

```bash
PC1> ping 20.25.37.2
```

*Resultado esperado:*

```
84 bytes from 20.25.37.2 icmp_seq=1 ttl=62 time=X ms
84 bytes from 20.25.37.2 icmp_seq=2 ttl=62 time=X ms
84 bytes from 20.25.37.2 icmp_seq=3 ttl=62 time=X ms
```

---

### 5.6 Tabla de comandos de verificación

| Comando | Qué muestra |
|---|---|
| `show interface Tunnel0` | Estado de la VTI: `up/up` indica túnel activo. |
| `show crypto isakmp sa` | Estado IKE Fase 1. Debe ser `QM_IDLE`. |
| `show crypto ipsec sa` | SAs de Fase 2, SPIs y contadores de paquetes. |
| `show ip route ospf` | Rutas OSPF aprendidas a través del túnel. |
| `show ip ospf neighbor` | Vecinos OSPF — confirma adyacencia sobre el VTI. |
| `show crypto ipsec profile` | Confirma el perfil IPSec vinculado al túnel. |
| `show crypto session` | Resumen del estado de todas las sesiones IPSec. |

---

## 6. Capturas de Pantalla

A continuación se detalla el índice de evidencias correspondientes a las fases de configuración, verificación y funcionamiento de la VPN Route-Based, alojadas en la carpeta [`screenshots/`](screenshots/README.md):

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_ping_sin_vpn.png`](screenshots/02_ping_sin_vpn.png) | Ping desde PC1 (`20.25.37.130`) hacia PC3 (`20.25.37.2`) **antes** de aplicar la configuración IPSec — demuestra conectividad WAN base. |
| 3 | [`03_config_r1_vti.png`](screenshots/03_config_r1_vti.png) | Consola de R1 mostrando la configuración de la interfaz `Tunnel0` con `tunnel mode ipsec ipv4` y `tunnel protection ipsec profile IPSEC_PROFILE_VTI`. |
| 4 | [`04_config_r1_isakmp_profile.png`](screenshots/04_config_r1_isakmp_profile.png) | Consola de R1 mostrando la ISAKMP policy, la PSK y el IPSec Profile configurado (sin Crypto Map). |
| 5 | [`05_config_r2_completa.png`](screenshots/05_config_r2_completa.png) | Configuración completa de R2 (Site B) mostrando la VTI con `tunnel destination 192.168.1.10` y el mismo IPSec Profile. |
| 6 | [`06_tunnel0_up.png`](screenshots/06_tunnel0_up.png) | Salida de `show interface Tunnel0` en R1 mostrando estado `up/up` y el protocolo de protección IPSec activo. |
| 7 | [`07_isakmp_sa_qmidle.png`](screenshots/07_isakmp_sa_qmidle.png) | Salida de `show crypto isakmp sa` mostrando estado `QM_IDLE` — Fase 1 completada. |
| 8 | [`08_ospf_route_aprendida.png`](screenshots/08_ospf_route_aprendida.png) | Salida de `show ip route ospf` en R1 mostrando la ruta `20.25.37.0/25` aprendida vía OSPF a través de `Tunnel0`. |
| 9 | [`09_ping_con_vpn_activa.png`](screenshots/09_ping_con_vpn_activa.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC4 (`20.25.37.3`) con el túnel IPSec activo, TTL=62. |

---

## 7. Consideraciones de Seguridad

### 7.1 Ventajas de seguridad del modelo Route-Based

* **Una única SA por túnel:** En el modelo *policy-based*, cada par de subredes genera su propio par de SAs IPSec. En la VTI, todo el tráfico comparte una sola SA, reduciendo la superficie de negociación y simplificando la auditoría.
* **Sin ACLs de tráfico interesante:** La eliminación de las ACLs de selección reduce el riesgo de misconfiguraciones que dejen tráfico sin cifrar por un error en la ACL.
* **Interfaz visible:** La VTI aparece en `show interface` y puede ser monitoreada como cualquier otro enlace, facilitando la detección de anomalías.

### 7.2 Hardening recomendado

```cisco
! Deshabilitar Aggressive Mode globalmente (aplica igual que en policy-based)
crypto isakmp aggressive-mode disable

! Eliminar la policy ISAKMP por defecto de Cisco (usa DES — inseguro)
no crypto isakmp policy 65535

! Forzar Perfect Forward Secrecy en el IPSec Profile
crypto ipsec profile IPSEC_PROFILE_VTI
 set pfs group14

! Aplicar un access-list en la interfaz WAN para
! permitir solo tráfico IKE y ESP desde el peer conocido
ip access-list extended ACL_WAN_IPSEC
 permit udp host 192.168.1.20 host 192.168.1.10 eq 500
 permit esp host 192.168.1.20 host 192.168.1.10
 deny ip any any log

interface Ethernet0/0
 ip access-group ACL_WAN_IPSEC in
```

### 7.3 Consideración sobre el MTU

La VTI agrega overhead de IPSec al paquete original. Con AES-256 y SHA-256 en modo túnel, el overhead típico es de aproximadamente 73–89 bytes. Por eso se configura:

* `ip mtu 1400` en la interfaz Tunnel0 para evitar fragmentación.
* `ip tcp adjust-mss 1360` para ajustar el MSS de las conexiones TCP y prevenir problemas con páginas web y transferencias de archivos.

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](#)**

**Duración:** máximo 8 minutos

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Demostración de los scripts de configuración aplicados en R1 y R2.
* ✅ Verificación de `show interface Tunnel0` mostrando estado `up/up`.
* ✅ Verificación de `show ip route ospf` mostrando la ruta aprendida dinámicamente a través del túnel.
* ✅ Verificación de `show crypto isakmp sa` mostrando estado `QM_IDLE`.
* ✅ Ping exitoso extremo a extremo entre PC1 (`20.25.37.130`) y PC3/PC4 (`20.25.37.2/3`).

---

## 9. Referencias

* Kent, S. & Seo, K. (2005). *RFC 4301 — Security Architecture for the Internet Protocol*. IETF.
* Harkins, D. & Carrel, D. (1998). *RFC 2409 — The Internet Key Exchange (IKEv1)*. IETF.
* Cisco Systems. (2024). *Cisco IOS Security Configuration Guide: Secure Connectivity — Virtual Tunnel Interfaces*.
* Cisco Systems. (2024). *Cisco IOS Security Command Reference — tunnel protection ipsec profile*.
* Doraswamy, N. & Harkins, D. (2003). *IPSec: The New Security Standard for the Internet, Intranets, and Virtual Private Networks (2nd Ed.)*. Prentice Hall.
