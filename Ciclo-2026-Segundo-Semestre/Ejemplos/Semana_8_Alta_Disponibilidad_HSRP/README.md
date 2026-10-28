# Semana 8 - Alta disponibilidad con HSRP

## Objetivo

Comprobar cómo dos routers comparten una puerta de enlace virtual y mantienen el acceso a un servidor cuando falla el router activo.

## Archivo y uso

Use **Cisco Packet Tracer 8.2.2 o una versión compatible**.

| Archivo | Uso |
|---|---|
| [Alta-Disponibilidad-HSRP-TOPOLOGIA.pkt](Alta-Disponibilidad-HSRP-TOPOLOGIA.pkt) | Equipos y cableado para resolver el ejercicio desde cero. |
| [Alta-Disponibilidad-HSRP.pkt](Alta-Disponibilidad-HSRP.pkt) | Ejemplo configurado para comprobar y comparar resultados. |

La versión `-TOPOLOGIA.pkt` conserva la distribución, los nombres visibles y los puertos utilizados. No contiene las IP, los protocolos ni los servicios personalizados del ejercicio; los dispositivos mantienen sus valores de fábrica. Los enlaces de los routers pueden aparecer desactivados hasta aplicar `no shutdown`.

Las tablas y comandos siguientes describen la versión configurada y permiten completar la topología base.

Trabaje en **Realtime** y espere a que los enlaces y protocolos converjan antes de probar. El primer ping puede perder paquetes por ARP o STP; repítalo para comprobar el estado estable.

## Subredes

| Segmento | Red/prefijo | Máscara | Hosts utilizables | Broadcast |
|---|---|---|---|---|
| LAN de clientes | `192.168.8.0/24` | `255.255.255.0` | `192.168.8.1` – `192.168.8.254` | `192.168.8.255` |
| Servicios | `192.168.80.0/24` | `255.255.255.0` | `192.168.80.1` – `192.168.80.254` | `192.168.80.255` |

## Direccionamiento de equipos

| Equipo | Interfaz | IP/prefijo | Gateway | DNS |
|---|---|---|---|---|
| R1 | GigabitEthernet0/0 | `192.168.8.2/24` | — | — |
| R1 | GigabitEthernet0/1 | `192.168.80.2/24` | — | — |
| R2 | GigabitEthernet0/0 | `192.168.8.3/24` | — | — |
| R2 | GigabitEthernet0/1 | `192.168.80.3/24` | — | — |
| SW-LAN | Puertos de capa 2 | Sin IP de administración | — | — |
| SW-SERVICIOS | Puertos de capa 2 | Sin IP de administración | — | — |
| PC1 | FastEthernet0 | `192.168.8.10/24` | 192.168.8.1 | — |
| PC2 | FastEthernet0 | `192.168.8.11/24` | 192.168.8.1 | — |
| WEB8 | FastEthernet0 | `192.168.80.10/24` | 192.168.80.1 | — |
| Grupo HSRP 8 | LAN virtual | `192.168.8.1/24` | Gateway de PC1 y PC2 | — |
| Grupo HSRP 80 | Servicios virtual | `192.168.80.1/24` | Gateway de WEB8 | — |

Las IP de clientes y servidores son estáticas. Se ingresan en **Desktop → IP Configuration**, junto con su máscara, gateway y DNS. Los switches conservan la VLAN 1 y conmutan en capa 2; no requieren IP para este ejemplo.

## Enlaces

| Origen | Destino | Cable |
|---|---|---|
| R1 GigabitEthernet0/0 | SW-LAN GigabitEthernet0/1 | Directo |
| R2 GigabitEthernet0/0 | SW-LAN GigabitEthernet0/2 | Directo |
| R1 GigabitEthernet0/1 | SW-SERVICIOS GigabitEthernet0/1 | Directo |
| R2 GigabitEthernet0/1 | SW-SERVICIOS GigabitEthernet0/2 | Directo |
| SW-LAN FastEthernet0/1 | PC1 FastEthernet0 | Directo |
| SW-LAN FastEthernet0/2 | PC2 FastEthernet0 | Directo |
| SW-SERVICIOS FastEthernet0/1 | WEB8 FastEthernet0 | Directo |

## Comandos de los routers

El archivo ya contiene la configuración. Estos bloques permiten reproducirla en una topología nueva con los mismos equipos y enlaces.

### R1

```cisco
enable
configure terminal
hostname R1
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 192.168.8.2 255.255.255.0
 no shutdown
 exit
interface GigabitEthernet0/1
 ip address 192.168.80.2 255.255.255.0
 no shutdown
 exit
interface GigabitEthernet0/0
standby 8 ip 192.168.8.1
standby 8 priority 110
standby 8 preempt
exit
interface GigabitEthernet0/1
standby 80 ip 192.168.80.1
standby 80 priority 110
standby 80 preempt
exit
end
write memory
```

### R2

```cisco
enable
configure terminal
hostname R2
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 192.168.8.3 255.255.255.0
 no shutdown
 exit
interface GigabitEthernet0/1
 ip address 192.168.80.3 255.255.255.0
 no shutdown
 exit
interface GigabitEthernet0/0
standby 8 ip 192.168.8.1
standby 8 priority 100
standby 8 preempt
exit
interface GigabitEthernet0/1
standby 80 ip 192.168.80.1
standby 80 priority 100
standby 80 preempt
exit
end
write memory
```

## Funcionamiento

- R1 tiene prioridad 110 y R2 prioridad 100 en ambos grupos.
- `standby 8 ip` define el gateway virtual de los clientes.
- `standby 80 ip` define el gateway virtual del servidor. Así el tráfico de retorno también tiene respaldo.
- `preempt` permite que R1 recupere el rol activo cuando vuelva a estar operativo.
- Los equipos finales usan las IP virtuales `.1`, no las IP físicas `.2` o `.3`.
- No se necesitan rutas estáticas: cada router está conectado a las dos subredes.

WEB8 tiene **Services → HTTP → On** y una página de identificación de la semana. Se puede abrir `http://192.168.80.10` desde los clientes.

## Verificación y conmutación

En ambos routers:

```cisco
show standby brief
show standby
show ip interface brief
```

En **PC1 → Desktop → Command Prompt**:

```text
ipconfig
ping 192.168.8.1
ping 192.168.80.10
```

1. Espere a que R1 aparezca `Active` y R2 `Standby` en los grupos 8 y 80.
2. Ejecute un ping al servidor para comprobar el estado inicial.
3. En **R1 → Physical**, apague el router completo. No cambie el gateway de los clientes.
4. Repita el ping y consulte `show standby brief` en R2: debe pasar a `Active` en ambos grupos.
5. Encienda R1 y espere la recuperación de enlaces y de HSRP. Por `preempt`, R1 volverá a `Active` y R2 a `Standby`.
6. Repita el ping y deje ambos routers encendidos.

| Prueba realizada en Packet Tracer 8.2.2 | Resultado observado |
|---|---|
| PC1 a WEB8 antes del fallo | 4/4 respuestas |
| Primera serie de pings después de apagar R1 | 2/4 respuestas; pérdida transitoria |
| Siguiente serie con R2 activo | 4/4 respuestas |
| Recuperación de R1 | Recuperó el estado Active en ambos grupos |

La pérdida depende del momento del fallo, ARP, STP y los temporizadores. Para medir el tiempo de conmutación, registre el tiempo de simulación al apagar R1 y al recibir la primera respuesta; no interprete el tiempo de ejecución de un comando como una medida exacta del protocolo.

Este ejercicio verifica la caída completa de un router. Una caída aislada de su enlace de salida requiere seguimiento de interfaces y un diseño que mantenga también el camino de retorno; ese seguimiento no está configurado aquí. Los switches y el servidor siguen siendo puntos únicos de fallo.

## Comparación con VRRP

| Aspecto | HSRP del ejemplo | VRRP |
|---|---|---|
| Roles | Active y Standby | Master y Backup |
| Gateway de los hosts | IP virtual compartida | IP virtual compartida |
| Elección | Prioridad; R1 tiene 110 | Prioridad; el propietario de la IP tiene 255 |
| Recuperación del preferido | `standby ... preempt` | Preempción según configuración y reglas del protocolo |

Se comprobó `vrrp ?` en el Cisco 2911 de Packet Tracer 8.2.2: el comando no está disponible. La práctica ejecutable utiliza HSRP; la comparación VRRP es conceptual. Un solo grupo HSRP tampoco distribuye automáticamente el tráfico entre los dos routers.
