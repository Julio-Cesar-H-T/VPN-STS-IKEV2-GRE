# VPN Site to Site con Túnel GRE sobre IPSec (IKEv2)

## Información del Proyecto

| Campo           | Detalle                               |
| --------------- | ------------------------------------- |
| **Estudiante**  | Julio César Hernández Tibrey          |
| **Matrícula**   | 2025-0702                             |
| **Institución** | Instituto Tecnológico de Las Americas |
| **Asignatura**  | Seguridad de Redes                    |
| **Profesor**    | Jonathan Esteban Rondón Corniel       |

---
## Objetivo

Establecer un túnel VPN site-to-site punto a punto entre **R1** (Sitio A) y **R2** (Sitio B) utilizando un **túnel GRE** protegido por **IPSec con IKEv2**. GRE encapsula el tráfico de usuario permitiendo soporte de multicast y routing dinámico; IKEv2 protege ese paquete GRE mediante un crypto map aplicado sobre la interfaz física pública, con un perfil IKEv2 explícito que reemplaza la negociación ISAKMP de IKEv1.

Este escenario es el más completo del laboratorio: combina la flexibilidad de GRE (multicast, OSPF/EIGRP) con la seguridad superior de IKEv2 (cookies anti-DoS, negociación en 4 mensajes, PSK por peer).

---

## Contexto: IKEv2 (Internet Key Exchange version 2)

- **Lanzamiento:** 2005 (RFC 4306), diseñado para corregir las deficiencias estructurales de IKEv1. Actualizado por RFC 7296 en 2014.
- **Eficiencia:** Reduce la negociación a solo **4 mensajes** (frente a los 6–9 de IKEv1 en Main Mode).
- **Seguridad mejorada:** Mecanismo de **cookies anti-DoS**. Elimina el vulnerable Aggressive Mode de IKEv1.
- **Autenticación asimétrica:** Cada peer puede autenticarse con un método distinto (PSK, certificado, EAP).
- **NAT Traversal nativo:** Sin necesidad de extensiones adicionales.
- **Estructura modular:** `proposal` + `policy` + `keyring` + `profile` — más granular y reutilizable que la `isakmp policy` de IKEv1.

---

## Comparativa IKEv1 vs IKEv2

| Aspecto | IKEv1 | IKEv2 |
|---------|-------|-------|
| RFC de referencia | 2409 (1998) | 7296 (2014) |
| Mensajes de negociación | 6–9 (Main) / 3 (Aggressive) | 4 siempre |
| Modo vulnerable | Aggressive Mode | No existe |
| NAT Traversal | Extensión opcional | Nativo |
| Protección DoS | Ninguna | Cookies IKEv2 |
| PSK en GRE | Global: `crypto isakmp key` | Por peer: dentro del `keyring` |
| Perfil en crypto map | No existe | `set ikev2-profile` |
| Comando de verificación | `show crypto isakmp sa` | `show crypto ikev2 sa` |
| Estado esperado | `QM_IDLE` | `READY` |

---

## Descripción: GRE sobre IPSec

**GRE (Generic Routing Encapsulation)** es un protocolo de tunelización que encapsula cualquier tipo de paquete para que pueda viajar entre dos puntos como si estuvieran en la misma red local.

### ¿Por qué GRE sobre IPSec con IKEv2?

GRE por sí solo **no proporciona encriptación**. Al combinarlo con IPSec IKEv2 se obtiene:
- **Flexibilidad de GRE:** Soporte para multicast, broadcast y protocolos de enrutamiento dinámico
- **Seguridad de IPSec IKEv2:** Encriptación, integridad y autenticación con el protocolo más moderno
- **PSK por peer:** La clave pre-compartida se define en el keyring por peer, no globalmente

### Características principales:
- Doble encapsulación: primero GRE encapsula el tráfico de usuario, luego IPSec cifra el paquete GRE completo
- El crypto map (con `set ikev2-profile`) se aplica sobre la interfaz física, no sobre Tunnel0
- La ACL de tráfico interesante protege el **protocolo GRE** entre las IPs públicas, no las subredes LAN
- Soporta enrutamiento dinámico y tráfico multicast sobre el túnel GRE

---

## Modo Transport vs Tunnel

| Aspecto | Mode Transport | Mode Tunnel |
|---------|----------------|--------------|
| Overhead | Menor | Mayor |
| Uso con GRE | **Recomendado** — GRE ya encapsula, IPSec solo agrega cabecera ESP | Redundante — doble cabecera IP |
| Qué protege IPSec | El payload del paquete GRE (cabeceras GRE + datos) | El paquete GRE completo con nueva cabecera IP |

> En este escenario se usa **`mode transport`**: GRE ya realiza el tunneling, IPSec en modo transporte solo añade su cabecera ESP sin re-encapsular el paquete IP, evitando overhead innecesario.

---

## Topología

![Topología VPN Site to Site GRE sobre IPSec IKEv2](img/topologia-vpn-gre-ikev2.png)

```
LAN A (10.7.2.0/27)                                  LAN B (10.7.2.32/27)
VLAN 10 / VLAN 20                                    VLAN 30 / VLAN 40
       |                                                    |
       R1 -- Tunnel0 172.16.12.1/30 (GRE) === Tunnel0 172.16.12.2/30 (GRE) -- R2
       |  e0/0 200.7.2.1/30 [crypto map] -- ISP -- [crypto map] 200.7.2.6/30 e0/1  |
```

### Tabla de Direccionamiento

| Dispositivo | Interfaz | Dirección IP | Máscara | VLAN/Descripción |
|-------------|----------|--------------|---------|-------------------|
| ISP | e0/0 | 200.7.2.2 | 255.255.255.252 | Enlace a R1 |
| ISP | e0/1 | 200.7.2.5 | 255.255.255.252 | Enlace a R2 |
| R1 | e0/0 | 200.7.2.1 | 255.255.255.252 | WAN — crypto map aplicado aquí (no en Tunnel0) |
| R1 | Tunnel0 | 172.16.12.1 | 255.255.255.252 | Túnel GRE |
| R1 | e0/1.10 | 10.7.2.1 | 255.255.255.240 | VLAN 10 - FINANZAS |
| R1 | e0/2.20 | 10.7.2.17 | 255.255.255.240 | VLAN 20 - RRHH |
| R2 | e0/1 | 200.7.2.6 | 255.255.255.252 | WAN — crypto map aplicado aquí (no en Tunnel0) |
| R2 | Tunnel0 | 172.16.12.2 | 255.255.255.252 | Túnel GRE |
| R2 | e0/0.30 | 10.7.2.33 | 255.255.255.240 | VLAN 30 - IT |
| R2 | e0/0.40 | 10.7.2.49 | 255.255.255.240 | VLAN 40 - ADMIN |

### Pools DHCP

**Sitio 1 (R1):**
| Pool | Red | Gateway | DNS |
|------|-----|---------|-----|
| FINANZAS | 10.7.2.0/28 | 10.7.2.1 | 8.8.8.8 |
| RRHH | 10.7.2.16/28 | 10.7.2.17 | 8.8.8.8 |

**Sitio 2 (R2):**
| Pool | Red | Gateway | DNS |
|------|-----|---------|-----|
| IT | 10.7.2.32/28 | 10.7.2.33 | 8.8.8.8 |
| ADMIN | 10.7.2.48/28 | 10.7.2.49 | 8.8.8.8 |

`[ESPACIO PARA CAPTURA: Topología completa en PNETLab con Tunnel0 GRE visible]`

---

## Configuración Base de la Infraestructura

### ISP

```
conf t
hostname ISP
int e0/0
 ip add 200.7.2.2 255.255.255.252
 no shut
int e0/1
 ip add 200.7.2.5 255.255.255.252
 no shut
exit
```

### R1 (Router on a Stick + DHCP)

```
conf t
hostname R1
int e0/0
 ip add 200.7.2.1 255.255.255.252
 no shut
exit
ip route 0.0.0.0 0.0.0.0 200.7.2.2
ip dhcp excluded-address 10.7.2.1
ip dhcp excluded-address 10.7.2.17
ip dhcp pool FINANZAS
 network 10.7.2.0 255.255.255.240
 default-router 10.7.2.1
 dns-server 8.8.8.8
ip dhcp pool RRHH
 network 10.7.2.16 255.255.255.240
 default-router 10.7.2.17
 dns-server 8.8.8.8
int e0/1
 no shut
int e0/1.10
 encapsulation dot1Q 10
 ip add 10.7.2.1 255.255.255.240
 no shut
int e0/2
 no shut
int e0/2.20
 encapsulation dot1Q 20
 ip add 10.7.2.17 255.255.255.240
 no shut
exit
```

### SW-1 (Acceso PC1 - VLAN 10)

```
conf t
hostname SW-1
vlan 10
 name FINANZAS
exit
int e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shut
int e0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shut
exit
```

### SW-2 (Acceso PC2 - VLAN 20)

```
conf t
hostname SW-2
vlan 20
exit
int e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shut
int e0/1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shut
exit
```

### R2 (Router on a Stick + DHCP)

```
conf t
hostname R2
int e0/1
 ip add 200.7.2.6 255.255.255.252
 no shut
exit
ip route 0.0.0.0 0.0.0.0 200.7.2.5
ip dhcp excluded-address 10.7.2.33
ip dhcp excluded-address 10.7.2.49
ip dhcp pool IT
 network 10.7.2.32 255.255.255.240
 default-router 10.7.2.33
 dns-server 8.8.8.8
ip dhcp pool ADMIN
 network 10.7.2.48 255.255.255.240
 default-router 10.7.2.49
 dns-server 8.8.8.8
int e0/0
 no shut
int e0/0.30
 encapsulation dot1Q 30
 ip add 10.7.2.33 255.255.255.240
 no shut
int e0/0.40
 encapsulation dot1Q 40
 ip add 10.7.2.49 255.255.255.240
 no shut
exit
```

### SW-3 (Trunk + Acceso PC3/PC4)

```
conf t
hostname SW-3
vlan 30
 name IT
vlan 40
 name ADMIN
exit
int e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shut
int e0/1
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shut
int e0/2
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shut
exit
```

### Verificación de la base

```
show ip interface brief
show vlan brief
show interfaces trunk
```

`[ESPACIO PARA CAPTURA: ping fallido entre LAN A y LAN B antes de aplicar la VPN]`

---

## Configuración GRE sobre IPSec con IKEv2

### Configuración R1

```
conf t
! ----- FASE 1: IKEv2 -----
crypto ikev2 proposal PROP-IKEV2
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy POL-IKEV2
 proposal PROP-IKEV2
exit

crypto ikev2 keyring KEY-IKEV2
 peer R2
  address 200.7.2.6
  pre-shared-key 2025-0702
 exit
exit

crypto ikev2 profile PROF-JULIO
 match identity remote address 200.7.2.6 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEY-IKEV2
exit

! ----- FASE 2: Transform Set en modo transport -----
crypto ipsec transform-set TS_GRE esp-aes 256 esp-sha256-hmac
 mode transport
exit

! ----- ACL: tráfico interesante es el protocolo GRE -----
ip access-list extended GRE-TRAFFIC
 permit gre host 200.7.2.1 host 200.7.2.6
exit

! ----- Crypto Map con IKEv2 Profile -----
crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 200.7.2.6
 set transform-set TS_GRE
 set ikev2-profile PROF-JULIO
 match address GRE-TRAFFIC
exit

! ----- Túnel GRE (sin cifrado propio) -----
interface Tunnel0
 ip address 172.16.12.1 255.255.255.252
 tunnel source e0/0
 tunnel destination 200.7.2.6
 tunnel mode gre ip
 no shutdown
exit

! ----- Crypto Map en la interfaz FÍSICA (no en Tunnel0) -----
interface e0/0
 crypto map CMAP-GRE
exit

ip route 10.7.2.32 255.255.255.224 Tunnel0
end
write memory
```

`[ESPACIO PARA CAPTURA: configuración completa aplicada en R1 — terminal]`

### Configuración R2

```
conf t
! ----- FASE 1: IKEv2 -----
crypto ikev2 proposal PROP-IKEV2
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy POL-IKEV2
 proposal PROP-IKEV2
exit

crypto ikev2 keyring KEY-IKEV2
 peer R1
  address 200.7.2.1
  pre-shared-key 2025-0702
 exit
exit

crypto ikev2 profile PROF-JULIO
 match identity remote address 200.7.2.1 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEY-IKEV2
exit

! ----- FASE 2: Transform Set en modo transport -----
crypto ipsec transform-set TS_GRE esp-aes 256 esp-sha256-hmac
 mode transport
exit

! ----- ACL: tráfico interesante es el protocolo GRE (espejo) -----
ip access-list extended GRE-TRAFFIC
 permit gre host 200.7.2.6 host 200.7.2.1
exit

! ----- Crypto Map con IKEv2 Profile -----
crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 200.7.2.1
 set transform-set TS_GRE
 set ikev2-profile PROF-JULIO
 match address GRE-TRAFFIC
exit

! ----- Túnel GRE -----
interface Tunnel0
 ip address 172.16.12.2 255.255.255.252
 tunnel source e0/1
 tunnel destination 200.7.2.1
 tunnel mode gre ip
 no shutdown
exit

! ----- Crypto Map en la interfaz FÍSICA -----
interface e0/1
 crypto map CMAP-GRE
exit

ip route 10.7.2.0 255.255.255.224 Tunnel0
end
write memory
```

`[ESPACIO PARA CAPTURA: configuración completa aplicada en R2 — terminal]`

---

## Explicación de los Componentes

### IKEv2 Proposal
| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `encryption aes-cbc-256` | AES-CBC 256 bits | Algoritmo de encriptación |
| `integrity sha256` | SHA-256 | Algoritmo de integridad |
| `group 14` | DH Group 14 (2048-bit) | Grupo Diffie-Hellman |

### IKEv2 Keyring
| Parámetro | Descripción |
|-----------|-------------|
| `peer R2` / `peer R1` | Nombre lógico del peer |
| `address` | IP pública del peer remoto |
| `pre-shared-key 2025-0702` | PSK definida por peer — ventaja de IKEv2 sobre IKEv1 |

### IKEv2 Profile
| Parámetro | Descripción |
|-----------|-------------|
| `match identity remote address` | Identifica el peer por IP pública (/32) |
| `authentication remote/local pre-share` | Ambos extremos se autentican con PSK |
| `keyring local KEY-IKEV2` | Referencia el keyring con la PSK del peer |

### Fase 2 (Transform Set)
| Parámetro | Descripción |
|-----------|-------------|
| `esp-aes 256` | Encriptación AES-256 |
| `esp-sha256-hmac` | Integridad SHA-256 |
| `mode transport` | GRE ya encapsula; IPSec solo protege sin nueva cabecera IP |

### ACL de tráfico interesante (protocolo GRE)
| Router | ACL | Regla |
|---|---|---|
| R1 | `GRE-TRAFFIC` | `permit gre host 200.7.2.1 host 200.7.2.6` |
| R2 | `GRE-TRAFFIC` | `permit gre host 200.7.2.6 host 200.7.2.1` |

> La ACL protege el **protocolo GRE** entre las IPs públicas, no las subredes LAN. El paquete GRE (que lleva el tráfico de usuario dentro) es lo que IPSec cifra.

### Crypto Map — diferencia con IKEv1 GRE
| Parámetro | IKEv1 GRE | IKEv2 GRE |
|---|---|---|
| Negociación | Usa `crypto isakmp policy` implícitamente | Referencia `set ikev2-profile PROF-JULIO` explícitamente |
| PSK | Global | Por peer en el keyring |

### Punto de aplicación
| Router | Interfaz donde se aplica el crypto map |
|---|---|
| R1 | e0/0 (física — **no** en Tunnel0) |
| R2 | e0/1 (física — **no** en Tunnel0) |

---

## Comparación entre los Seis Escenarios

| Aspecto | Policy IKEv1 | Route IKEv1 | GRE IKEv1 | Policy IKEv2 | Route IKEv2 | GRE IKEv2 |
|---|---|---|---|---|---|---|
| Protocolo IKE | IKEv1 | IKEv1 | IKEv1 | IKEv2 | IKEv2 | IKEv2 |
| Tipo de túnel | Ninguno | VTI IPSec | GRE | Ninguno | VTI IPSec | GRE |
| Modo transform-set | Tunnel | Tunnel | Transport | Tunnel | Tunnel | Transport |
| ACL requerida | Sí (subredes LAN) | No | Sí (GRE) | Sí (subredes LAN) | No | Sí (GRE) |
| Multicast / dyn. routing | No | No | Sí | No | No | **Sí** |
| PSK por peer | No | No | No | Sí | Sí | **Sí** |
| Mensajes de negociación | 6–9 | 6–9 | 6–9 | 4 | 4 | **4** |
| Verificación SA | `isakmp sa` | `isakmp sa` | `isakmp sa` | `ikev2 sa` | `ikev2 sa` | `ikev2 sa` |

---

## Verificación

**Estado del túnel GRE:**
```
show interface tunnel 0
```

`[ESPACIO PARA CAPTURA: show interface tunnel0 — up/up, encapsulación GRE]`

**Estado del túnel IKEv2:**
```
show crypto ikev2 sa
```

`[ESPACIO PARA CAPTURA: show crypto ikev2 sa — estado READY]`

**Explicación:** El estado `READY` confirma que la negociación IKEv2 de 4 mensajes se completó correctamente entre 200.7.2.1 y 200.7.2.6.

**Estado de las SAs IPSec:**
```
show crypto ipsec sa
```

`[ESPACIO PARA CAPTURA: show crypto ipsec sa — local/remote ident con protocolo GRE]`

**Explicación:** El `local ident`/`remote ident` debe mostrar el **protocolo GRE** entre las IPs públicas — no subredes LAN ni IP genérico. Esto confirma que IPSec cifra específicamente los paquetes GRE.

**Sesión de cifrado:**
```
show crypto session
```

`[ESPACIO PARA CAPTURA: show crypto session — UP-ACTIVE]`

**Crypto map en interfaz física:**
```
show crypto map
```

`[ESPACIO PARA CAPTURA: show crypto map — aplicado en e0/0 (R1) y e0/1 (R2), no en Tunnel0]`

**Prueba de conectividad entre extremos del túnel:**
```
ping 172.16.12.2 source tunnel0
```

`[ESPACIO PARA CAPTURA: ping exitoso entre IPs de túnel GRE]`

**Prueba de conectividad entre LANs:**
```
ping 10.7.2.34
traceroute 10.7.2.34 source e0/1.10
```

`[ESPACIO PARA CAPTURA: ping y traceroute exitoso entre LAN A y LAN B]`

---

## Notas y Consideraciones

- Este escenario es el más completo del laboratorio: GRE + IPSec IKEv2 combina soporte de multicast/routing dinámico con la mayor seguridad disponible en el stack de VPN de Cisco IOS.
- A diferencia del GRE con IKEv1 (donde la PSK es global), IKEv2 define la PSK por peer en el keyring, facilitando la gestión en topologías con múltiples sitios remotos.
- El `show crypto isakmp sa` no aplica en IKEv2 — siempre usar `show crypto ikev2 sa`.
- Nombres personalizados con la matrícula del autor (`PROF-JULIO`, `2025-0702`) para trazabilidad del laboratorio.
