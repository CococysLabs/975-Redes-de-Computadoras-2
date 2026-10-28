# Semana 11 - RIP, BGP, TLS y redes en la nube

## Objetivo

Representar una sede privada conectada a dos subredes de servicios de un proveedor simulado. Comparar RIP y BGP y analizar disponibilidad web, TLS y balanceo de carga.

## Archivo y uso

Use **Cisco Packet Tracer 8.2.2 o una versión compatible**.

| Archivo | Uso |
|---|---|
| [RIP-BGP-Nube-TLS-TOPOLOGIA.pkt](RIP-BGP-Nube-TLS-TOPOLOGIA.pkt) | Equipos y cableado para resolver el ejercicio desde cero. |
| [RIP-BGP-Nube-TLS.pkt](RIP-BGP-Nube-TLS.pkt) | Ejemplo configurado para comprobar y comparar resultados. |

La versión `-TOPOLOGIA.pkt` conserva la distribución, los nombres visibles y los puertos utilizados. No contiene las IP, los protocolos ni los servicios personalizados del ejercicio; los dispositivos mantienen sus valores de fábrica. Los enlaces de los routers pueden aparecer desactivados hasta aplicar `no shutdown`.

Las tablas y comandos siguientes describen la versión configurada y permiten completar la topología base.

Trabaje en **Realtime** y espere a que los enlaces y protocolos converjan antes de probar. El primer ping puede perder paquetes por ARP o STP; repítalo para comprobar el estado estable.

**Alcance:** conectividad RIP/eBGP, dos subredes de servidores, DNS y HTTPS simulado. TLS real, balanceo de carga y elasticidad se trabajan como análisis conceptual al final.

## Subredes

| Segmento | Red/prefijo | Máscara | Hosts utilizables | Broadcast |
|---|---|---|---|---|
| R-SEDE a R-BORDE | `10.11.0.0/30` | `255.255.255.252` | `10.11.0.1` – `10.11.0.2` | `10.11.0.3` |
| R-BORDE a R-NUBE | `10.11.0.4/30` | `255.255.255.252` | `10.11.0.5` – `10.11.0.6` | `10.11.0.7` |
| LAN privada | `10.11.10.0/24` | `255.255.255.0` | `10.11.10.1` – `10.11.10.254` | `10.11.10.255` |
| Zona A del proveedor | `172.20.11.0/26` | `255.255.255.192` | `172.20.11.1` – `172.20.11.62` | `172.20.11.63` |
| Zona B del proveedor | `172.20.11.64/26` | `255.255.255.192` | `172.20.11.65` – `172.20.11.126` | `172.20.11.127` |

## Direccionamiento de equipos

| Equipo | Interfaz | IP/prefijo | Gateway | DNS |
|---|---|---|---|---|
| R-SEDE | GigabitEthernet0/0 | `10.11.0.1/30` | — | — |
| R-SEDE | GigabitEthernet0/1 | `10.11.10.1/24` | — | — |
| R-BORDE | GigabitEthernet0/0 | `10.11.0.2/30` | — | — |
| R-BORDE | GigabitEthernet0/1 | `10.11.0.5/30` | — | — |
| R-NUBE | GigabitEthernet0/0 | `10.11.0.6/30` | — | — |
| R-NUBE | GigabitEthernet0/1 | `172.20.11.1/26` | — | — |
| R-NUBE | GigabitEthernet0/2 | `172.20.11.65/26` | — | — |
| SW-SEDE | Puertos de capa 2 | Sin IP de administración | — | — |
| PC-SEDE | FastEthernet0 | `10.11.10.10/24` | 10.11.10.1 | 172.20.11.11 |
| SW-ZONA-A | Puertos de capa 2 | Sin IP de administración | — | — |
| SW-ZONA-B | Puertos de capa 2 | Sin IP de administración | — | — |
| WEB-A | FastEthernet0 | `172.20.11.10/26` | 172.20.11.1 | — |
| DNS-NUBE | FastEthernet0 | `172.20.11.11/26` | 172.20.11.1 | — |
| WEB-B | FastEthernet0 | `172.20.11.74/26` | 172.20.11.65 | — |

Las IP de clientes y servidores son estáticas. Se ingresan en **Desktop → IP Configuration**, junto con su máscara, gateway y DNS. Los switches conservan la VLAN 1 y conmutan en capa 2; no requieren IP para este ejemplo.

## Enlaces

| Origen | Destino | Cable |
|---|---|---|
| R-SEDE GigabitEthernet0/0 | R-BORDE GigabitEthernet0/0 | Cruzado |
| R-BORDE GigabitEthernet0/1 | R-NUBE GigabitEthernet0/0 | Cruzado |
| R-SEDE GigabitEthernet0/1 | SW-SEDE GigabitEthernet0/1 | Directo |
| SW-SEDE FastEthernet0/1 | PC-SEDE FastEthernet0 | Directo |
| R-NUBE GigabitEthernet0/1 | SW-ZONA-A GigabitEthernet0/1 | Directo |
| R-NUBE GigabitEthernet0/2 | SW-ZONA-B GigabitEthernet0/1 | Directo |
| SW-ZONA-A FastEthernet0/1 | WEB-A FastEthernet0 | Directo |
| SW-ZONA-A FastEthernet0/2 | DNS-NUBE FastEthernet0 | Directo |
| SW-ZONA-B FastEthernet0/1 | WEB-B FastEthernet0 | Directo |

## Comandos de los routers

El archivo ya contiene la configuración. Estos bloques permiten reproducirla en una topología nueva con los mismos equipos y enlaces.

### R-SEDE

```cisco
enable
configure terminal
hostname R-SEDE
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 10.11.0.1 255.255.255.252
 no shutdown
 exit
interface GigabitEthernet0/1
 ip address 10.11.10.1 255.255.255.0
 no shutdown
 exit
router rip
 version 2
 no auto-summary
 network 10.0.0.0
passive-interface GigabitEthernet0/1
exit
end
write memory
```

### R-BORDE

```cisco
enable
configure terminal
hostname R-BORDE
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 10.11.0.2 255.255.255.252
 no shutdown
 exit
interface GigabitEthernet0/1
 ip address 10.11.0.5 255.255.255.252
 no shutdown
 exit
ip route 0.0.0.0 0.0.0.0 10.11.0.6
router rip
 version 2
 no auto-summary
 network 10.0.0.0
passive-interface GigabitEthernet0/1
default-information originate
exit
router bgp 65110
neighbor 10.11.0.6 remote-as 65120
network 10.11.10.0 mask 255.255.255.0
no auto-summary
exit
end
write memory
```

### R-NUBE

```cisco
enable
configure terminal
hostname R-NUBE
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 10.11.0.6 255.255.255.252
 no shutdown
 exit
interface GigabitEthernet0/1
 ip address 172.20.11.1 255.255.255.192
 no shutdown
 exit
interface GigabitEthernet0/2
 ip address 172.20.11.65 255.255.255.192
 no shutdown
 exit
router bgp 65120
neighbor 10.11.0.5 remote-as 65110
network 172.20.11.0 mask 255.255.255.192
network 172.20.11.64 mask 255.255.255.192
no auto-summary
exit
end
write memory
```

## Relación entre RIP y BGP

| Tramo | Protocolo | Propósito |
|---|---|---|
| R-SEDE ↔ R-BORDE | RIPv2 | Aprender las redes internas del AS 65110 |
| R-BORDE ↔ R-NUBE | eBGP | Intercambiar redes entre AS 65110 y 65120 |
| R-BORDE hacia R-SEDE | Ruta por defecto anunciada por RIP | Dar salida a la sede |

R-BORDE aprende `10.11.10.0/24` por RIP y la anuncia mediante `network` en BGP. R-NUBE anuncia las dos subredes `/26`. R-BORDE tiene una ruta estática por defecto al proveedor y usa `default-information originate` para anunciarla a la sede mediante RIP. Las rutas BGP más específicas se usan para las zonas A y B.

No se utiliza redistribución directa BGP→RIP, que no está disponible en la CLI de esta imagen. `passive-interface` evita enviar anuncios RIP hacia la LAN de clientes o hacia el proveedor. `network 10.0.0.0` selecciona las interfaces de esa red mayor; `no auto-summary` conserva las máscaras específicas.

RIP usa saltos, un máximo de 15 alcanzables y UDP 520. BGP intercambia atributos y usa TCP 179. No son reemplazos equivalentes: en este ejemplo cumplen funciones diferentes.

## Servicios y DNS

En **DNS-NUBE → Services → DNS**, active el servicio y configure:

| Nombre | Tipo | IP |
|---|---|---|
| `app.redes2.test` | A | `172.20.11.10` |
| `zona-a.redes2.test` | A | `172.20.11.10` |
| `zona-b.redes2.test` | A | `172.20.11.74` |

En WEB-A y WEB-B, deje **Services → HTTP → HTTP On / HTTPS On**. Cada `index.html` identifica su zona. DNS-NUBE tiene HTTP y HTTPS desactivados. PC-SEDE usa `172.20.11.11` como DNS.

Las dos zonas son subredes de laboratorio. Tener dos subredes no demuestra aislamiento de fallos físico ni equivale a desplegar dos zonas de disponibilidad reales. No hay conexión a un proveedor externo ni recursos cloud creados.

## Verificación

En R-SEDE:

```cisco
show ip route
show ip protocols
```

En R-BORDE y R-NUBE:

```cisco
show ip bgp summary
show ip bgp
show ip route
```

En PC-SEDE:

```text
ipconfig /all
ping 172.20.11.10
ping 172.20.11.74
nslookup app.redes2.test
nslookup zona-b.redes2.test
tracert 172.20.11.74
```

Desde **Web Browser**:

```text
https://app.redes2.test
https://zona-a.redes2.test
https://zona-b.redes2.test
```

| Comprobación realizada | Resultado observado |
|---|---|
| R-SEDE | Ruta `R* 0.0.0.0/0` vía `10.11.0.2` |
| R-BORDE | LAN `10.11.10.0/24` aprendida por RIP; ambas zonas `/26` por BGP |
| R-NUBE | `10.11.10.0/24` aprendida por BGP vía `10.11.0.5` |
| Ping de PC-SEDE a WEB-A y WEB-B | 4/4 respuestas a cada servidor después de ARP |
| DNS inicial | `app.redes2.test` → `172.20.11.10` |
| HTTPS a ambas zonas | Página identificada como Zona A y Zona B, respectivamente |

## Ejercicio de recuperación manual

1. Apague WEB-A desde su pestaña **Physical**.
2. Compruebe que WEB-B continúa accesible usando `zona-b.redes2.test`.
3. En DNS-NUBE, edite el registro A de `app.redes2.test` para apuntar a `172.20.11.74`.
4. Ejecute `nslookup app.redes2.test` y compruebe el cambio. Las cachés del cliente o del navegador pueden conservar la respuesta anterior; cierre y reabra el navegador o espere su renovación.
5. Vuelva a encender WEB-A y restaure el registro a `172.20.11.10`.

Se comprobó el cambio de resolución a `.74` con WEB-A apagado y se restauró el valor inicial. Es una intervención manual sobre DNS: no es balanceo de carga ni detección automática de fallos.

## Diseño conceptual de balanceo y TLS

El `.pkt` implementa conectividad y acceso directo a dos servidores. Para el caso de comercio electrónico de la presentación, la arquitectura adicional sería:

```text
Cliente --HTTPS--> Balanceador --HTTP/HTTPS--> WEB-A / WEB-B
                         |
                 Comprobaciones de salud
```

| Elemento propuesto | Función |
|---|---|
| Nombre público de la aplicación | Apuntar al punto de entrada del balanceador |
| Listener HTTPS | Presentar el certificado y terminar TLS en el punto de entrada |
| Dos destinos web | Atender solicitudes repartidas por el balanceador |
| Comprobación de salud | Retirar temporalmente un destino que no responde correctamente |
| Escalado | Agregar o retirar capacidad según métricas; requiere un mecanismo adicional |

En este archivo no hay balanceador, health checks automáticos, virtualización, Auto Scaling ni emisión de certificados. El servicio HTTPS se simula en cada Server-PT. El despliegue propuesto requeriría una plataforma que implemente esos componentes.

Para analizar certificados, revise nombre, vigencia, firma y cadena hasta una CA de confianza. El handshake acuerda parámetros y claves; después se protegen los datos con cifrado simétrico. La simulación no valida certificados públicos, cadenas reales ni una versión concreta de TLS.

Compare los modelos de nube: pública (recursos de proveedor), privada (uso exclusivo de una organización) e híbrida (integración de infraestructuras). Un esquema SaaS también necesita aislamiento de datos y permisos por cliente; separar subredes o usar distintos nombres DNS no lo garantiza por sí solo.
