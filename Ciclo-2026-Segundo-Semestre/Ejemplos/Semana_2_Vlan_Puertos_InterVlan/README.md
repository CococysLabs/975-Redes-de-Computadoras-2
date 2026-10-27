# Clase 1

# VLANs

| NO  | NOMBRE         | **Id Red**   | **Mascara de red** | **Puerta de enlace predeterminada** |
| --- | -------------- | ------------ | ------------------ | ----------------------------------- |
| 10  | ventas         | 192.168.10.0 | /25                | 192.168.10.1                        |
| 20  | secretaria     | 192.168.20.0 | /25                | 192.168.20.2                        |
| 30  | administracion | 192.168.30.0 | /25                | 192.168.30.3                        |
| 40  | juridico       | 192.168.40.0 | /25                | 192.168.40.4                        |

# **Tabla de IP's**

| **Equipo** | **Vlan**       | **IP**       | **Mascara de red** |
| ---------- | -------------- | ------------ | ------------------ |
| PC0        | ventas         | 192.168.10.2 | 255.255.255.0      |
| PC1        | ventas         | 192.168.10.3 | 255.255.255.0      |
| PC2        | secretaria     | 192.168.20.2 | 255.255.255.0      |
| PC3        | administracion | 192.168.30.2 | 255.255.255.0      |
| PC4        | juridico       | 192.168.40.2 | 255.255.255.0      |
| PC5        | juridico       | 192.168.40.3 | 255.255.255.0      |

# Configuraciones

## Switch0

```bash
enable

conf t

! Configuración de vtp
vtp mode client
vtp version 2
vtp domain clase1

! Puertos troncales
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 exit

! Puertos de acceso
interface FastEthernet0/11
 switchport access vlan 10
 switchport mode access
 exit

interface FastEthernet0/12
 switchport access vlan 20
 switchport mode access
 exit

do wr
```

## Switch1

```bash
enable

conf t

! Configuración de vtp
vtp mode server
vtp version 2
vtp domain clase1

!Creación de Vlans
vlan 10
name ventas
vlan 20
name secretaria

! Puertos troncales
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 exit

do wr

interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 exit

do wr
```

## Router0

```bash
enable

conf t

interface GigabitEthernet0/0
 no shutdown

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 exit

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 exit

do wr
```

## Switch2

```bash
enable

conf t

! Configuración de vtp
vtp mode server
vtp domain admin

!Creación de Vlans
vlan 30
name administracion
vlan 40
name juridico

! Puertos troncales
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 exit

! Puertos de Acceso

interface FastEthernet0/10
 switchport access vlan 30
 switchport mode access
 exit

interface FastEthernet0/11
 switchport access vlan 40
 switchport mode access
 exit

interface FastEthernet0/12
 switchport access vlan 40
 switchport mode access
 exit

do wr
```

## Multilayer Switch0

```bash
enable

conf t

! Necesario para que el swicht actue como router
ip routing

! Puertos troncales
interface gigabitEthernet 0/1
switchport trunk encapsulation dot1q
no shutdown
exit

! Configuración de vtp
vtp mode client
vtp domain admin

interface vlan 30
no shutdown
ip address 192.168.30.1 255.255.255.0
exit

interface vlan 40
no shutdown
ip address 192.168.40.1 255.255.255.0
exit

do wr
```
