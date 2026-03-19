# VPN Site-To-Site — Documentación Técnica

**Estudiante:** Reily Castillo  
**Matrícula:** 20241198  
**Asignatura:** Seguridad de Redes  
**Institución:** ITLA  

---

## 1. Objetivo de la Red

Implementar una VPN Site-To-Site entre dos sitios remotos utilizando dos firewalls FortiGate conectados a través de un router ISP simulado en GNS3. El objetivo es establecer un túnel IPSec cifrado que permita la comunicación segura entre las redes LAN de ambos sitios, verificando la conectividad mediante ping y traceroute.

---

## 2. Topología

```
 PC1 (20.24.1.10)                              PC2 (20.24.2.10)
       |                                               |
  [FortiGate-1]                               [FortiGate-2]
  LAN:  20.24.1.1/24  ←—— Túnel IPSec ——→  LAN:  20.24.2.1/24
  WAN:  11.98.0.1/30                         WAN:  11.98.1.2/30
       |                                               |
      [R1 — Router ISP]
    g0/0: 11.98.0.2/30
    e1/0: 11.98.1.1/30
```

### Dispositivos utilizados

| Dispositivo | Rol | Plataforma |
|---|---|---|
| FortiGate-1 | Firewall Sitio A | FortiGate VM 7.0.9 |
| FortiGate-2 | Firewall Sitio B | FortiGate VM 7.0.9 |
| R1 | Router ISP (tránsito) | Cisco IOU |
| PC1 | Host Sitio A | VPCS |
| PC2 | Host Sitio B | VPCS |
| Cloud1 | Acceso a Internet | GNS3 NAT Cloud |

---

## 3. Direccionamiento IP

### FortiGate-1 (Sitio A)

| Interfaz | Alias | IP / Máscara | Rol |
|---|---|---|---|
| port1 | LAN-A | 20.24.1.1 / 255.255.255.0 | LAN Sitio A |
| port2 | WAN | 11.98.0.1 / 255.255.255.252 | WAN hacia ISP |

### FortiGate-2 (Sitio B)

| Interfaz | Alias | IP / Máscara | Rol |
|---|---|---|---|
| port1 | WAN | 11.98.1.2 / 255.255.255.252 | WAN hacia ISP |
| port2 | LAN-B | 20.24.2.1 / 255.255.255.0 | LAN Sitio B |

### Router ISP (R1)

| Interfaz | IP / Máscara | Conectado a |
|---|---|---|
| g0/0 | 11.98.0.2 / 255.255.255.252 | FortiGate-1 WAN |
| e1/0 | 11.98.1.1 / 255.255.255.252 | FortiGate-2 WAN |

### Hosts

| Host | IP | Gateway | Red |
|---|---|---|---|
| PC1 | 20.24.1.10 | 20.24.1.1 | Sitio A |
| PC2 | 20.24.2.10 | 20.24.2.1 | Sitio B |

---

## 4. Configuración del Túnel VPN IPSec

### 4.1 FortiGate-1 — Fase 1 (IKE)

```
config vpn ipsec phase1-interface
    edit "VPN-TO-FGT2"
        set interface "port2"
        set remote-gw 11.98.1.2
        set psksecret <clave-compartida>
        set ike-version 2
    next
end
```

### 4.2 FortiGate-1 — Fase 2 (IPSec)

```
config vpn ipsec phase2-interface
    edit "VPN-TO-FGT2-P2"
        set phase1name "VPN-TO-FGT2"
        set src-subnet 20.24.1.0 255.255.255.0
        set dst-subnet 20.24.2.0 255.255.255.0
    next
end
```

### 4.3 FortiGate-2 — Fase 1 (IKE)

```
config vpn ipsec phase1-interface
    edit "VPN-TO-FGT1"
        set interface "port1"
        set remote-gw 11.98.0.1
        set psksecret <clave-compartida>
        set ike-version 2
    next
end
```

### 4.4 FortiGate-2 — Fase 2 (IPSec)

```
config vpn ipsec phase2-interface
    edit "VPN-TO-FGT1-P2"
        set phase1name "VPN-TO-FGT1"
        set src-subnet 20.24.2.0 255.255.255.0
        set dst-subnet 20.24.1.0 255.255.255.0
    next
end
```

---

## 5. Rutas Estáticas

### FortiGate-1

```
config router static
    edit 1
        set dst 20.24.2.0 255.255.255.0
        set device "VPN-TO-FGT2"
    next
end
```

### FortiGate-2

```
config router static
    edit 1
        set dst 20.24.1.0 255.255.255.0
        set device "VPN-TO-FGT1"
    next
end
```

---

## 6. Políticas de Firewall

En ambos FortiGates se configuraron dos políticas para permitir el tráfico bidireccional a través del túnel VPN:

| Política | Src Interface | Dst Interface | Acción |
|---|---|---|---|
| LAN-to-VPN | LAN (port1) | VPN-TO-FGT2 | ACCEPT |
| VPN-to-LAN | VPN-TO-FGT2 | LAN (port1) | ACCEPT |

### Configuración CLI (FortiGate-1)

```
config firewall policy
    edit 1
        set name "LAN-to-VPN"
        set srcintf "LAN-A"
        set dstintf "VPN-TO-FGT2"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        set action accept
        set nat disable
    next
    edit 2
        set name "VPN-to-LAN"
        set srcintf "VPN-TO-FGT2"
        set dstintf "LAN-A"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        set action accept
    next
end
```

---

## 7. Estado del Túnel VPN

El túnel `VPN-TO-FGT2` aparece con **Status: Up** en la sección VPN > IPSec Tunnels del FortiGate-1, confirmando que la negociación IKE fue exitosa y el túnel está activo.

---

## 8. Verificación de Conectividad

### Ping desde PC1 hacia PC2

```
PC1> ping 20.24.2.10
84 bytes from 20.24.2.10  icmp_seq=1 ttl=62 time=14.951 ms
84 bytes from 20.24.2.10  icmp_seq=2 ttl=62 time=18.512 ms
84 bytes from 20.24.2.10  icmp_seq=3 ttl=62 time=15.905 ms
84 bytes from 20.24.2.10  icmp_seq=4 ttl=62 time=22.331 ms
```

✅ Ping exitoso — comunicación entre sitios establecida.

### Traceroute desde PC1 hacia PC2

```
PC1> trace 20.24.2.10
trace to 20.24.2.10, 8 hops max

1   20.24.1.1    0.763 ms   0.696 ms   0.593 ms   ← FortiGate-1 LAN
2   11.98.1.2   16.766 ms  19.513 ms  20.192 ms   ← FortiGate-2 WAN
3  *20.24.2.10  20.530 ms  (ICMP type:3, code:3)  ← PC2 destino
```

### Análisis del Traceroute

| Salto | IP | Dispositivo | Descripción |
|---|---|---|---|
| 1 | 20.24.1.1 | FortiGate-1 | Gateway LAN Sitio A |
| 2 | 11.98.1.2 | FortiGate-2 WAN | Extremo remoto del túnel VPN |
| 3 | 20.24.2.10 | PC2 | Destino final Sitio B |

El traceroute confirma que el tráfico viaja directamente a través del túnel IPSec desde FortiGate-1 a FortiGate-2, sin pasar por el router ISP como salto visible — lo cual es el comportamiento esperado en una VPN Site-To-Site.

---

## 9. Conclusión

La VPN Site-To-Site fue configurada exitosamente utilizando dos FortiGate VM 7.0.9 en GNS3. El túnel IPSec con IKEv2 permite comunicación cifrada y transparente entre las redes LAN de ambos sitios (20.24.1.0/24 y 20.24.2.0/24). La conectividad fue verificada mediante ping y traceroute, comprobando el correcto funcionamiento del enlace VPN.

---

*Documentación generada para la asignatura Seguridad de Redes — ITLA*  
*Estudiante: Reily Castillo | Matrícula: 20241198*
