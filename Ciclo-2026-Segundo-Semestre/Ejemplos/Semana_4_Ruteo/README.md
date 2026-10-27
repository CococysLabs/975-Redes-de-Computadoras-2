# Clase 2

Clase 2.pkt

# tabla

| **Equipos** | **Id de Red** | **Mascara de red**  |
| ----------- | ------------- | ------------------- |
| R0 – R1     | 10.0.1.0      | 255.255.255.0 (/24) |
| R1 – R2     | 10.0.2.0      | 255.255.255.0 (/24) |
| R2 – MLS1   | 10.0.3.0      | 255.255.255.0 (/24) |
| VLAN 10     | 192.168.10.0  | 255.255.255.0 (/24) |
| VLAN 20     | 192.168.20.0  | 255.255.255.0 (/24) |
| VLAN 30     | 192.168.30.0  | 255.255.255.0 (/24) |
| VLAN 40     | 192.168.40.0  | 255.255.255.0 (/24) |

## R0

```bash
enable
conf t
interface GigabitEthernet0/1
 ip address 10.0.1.1 255.255.255.0
 no shutdown
 exit
```

```bash
router eigrp 1
 network 10.0.1.0 0.0.0.255
 network 192.168.10.0
 network 192.168.20.0
```

## R1

## Descripcion

En el Router 1 se configuran dos interfaces físicas que permiten la interconexión entre el dominio EIGRP (R0) y el dominio RIP (R2). Este router actúa como punto de redistribución entre ambos protocolos.

```bash
interface GigabitEthernet0/0
 ip address 10.0.1.2 255.255.255.0
 no shutdown
 exit
interface GigabitEthernet0/1
 ip address 10.0.2.1 255.255.255.0
 no shutdown
 exit
```

## Descripcion

Configuración del proceso EIGRP y redistribución de rutas provenientes de RIP.

```bash
router eigrp 1
 redistribute rip metric 10000 0 255 1 1500
 network 10.0.1.0 0.0.0.255
```

Parámetros de redistribute rip metric 10000 0 255 1 1500

Cuando se redistribuyen rutas hacia EIGRP es obligatorio definir una métrica compuesta, ya que EIGRP utiliza múltiples parámetros para calcular su métrica:

- 10000 → Ancho de banda (Bandwidth) en Kbps
- 0 → Delay acumulado
- 255 → Confiabilidad (Reliability)
- 1 → Carga (Load)
- 1500 → MTU (Maximum Transmission Unit)
  Estos valores permiten que EIGRP pueda calcular correctamente el costo de las rutas aprendidas desde RIP.

# Descripcion

Configuración del proceso RIP y redistribución de rutas provenientes de EIGRP.

```bash
router rip
 version 2
 redistribute eigrp 1 metric 2
 network 10.0.0.0
```

Parámetros de redistribute eigrp 1 metric 2

- 1 → Número del proceso EIGRP del cual se redistribuyen las rutas.
- metric 2 → Define el número de saltos (hop count) con el que RIP anunciará las rutas externas.

RIP utiliza únicamente el conteo de saltos como métrica, por lo tanto es obligatorio asignar un valor manual cuando se redistribuyen rutas desde otro protocolo.

## R2

```bash
interface GigabitEthernet0/0
 ip address 10.0.2.2 255.255.255.0
 no shutdown
 exit
interface GigabitEthernet0/1
 ip address 10.0.3.1 255.255.255.0
 no shutdown
 exit
```

# Descripcion

Configuración del proceso OSPF y redistribución de rutas provenientes de RIP.

```bash
router ospf 1
 redistribute rip subnets
 network 10.0.3.0 0.0.0.255 area 0
```

Parámetros de redistribute rip subnets

- rip → Indica que se redistribuyen rutas provenientes del proceso RIP.
- subnets → Permite que OSPF anuncie todas las subredes, no solo las redes claseful.

En OSPF no es obligatorio definir métrica manual en este caso, ya que asigna automáticamente un costo por defecto a las rutas redistribuidas (E2 por defecto).

# Descripcion

Configuración de RIP en R2 y redistribución de rutas provenientes de OSPF.

```bash
router rip
 version 2
 redistribute ospf 1 metric 2
 network 10.0.0.0
 no auto-summary
```

Parámetros de redistribute ospf 1 metric 2

- 1 → Número del proceso OSPF del cual se redistribuyen rutas.
- metric 2 → Define el número de saltos con el que RIP anunciará las rutas externas.

## Multilayer Switch0

```bash
interface GigabitEthernet0/2
 no switchport
 ip address 10.0.3.2 255.255.255.0
 exit
```

# Descripción

En el Multilayer Switch se habilita el enrutamiento dinámico mediante OSPF. Este dispositivo anuncia las redes correspondientes a las VLAN 30 y VLAN 40, permitiendo la comunicación inter-VLAN y su propagación hacia el resto de la topología.

```bash
router ospf 1
 log-adjacency-changes
 network 10.0.3.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.40.0 0.0.0.255 area 0
```
