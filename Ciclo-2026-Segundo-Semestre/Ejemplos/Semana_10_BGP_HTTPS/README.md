# Semana 10 - BGP y HTTPS

## Objetivo

Interconectar dos sistemas autónomos mediante eBGP y acceder a un servidor DNS y HTTP/HTTPS desde la red remota.

## Archivo y uso

Use **Cisco Packet Tracer 8.2.2 o una versión compatible**.

| Archivo | Uso |
|---|---|
| [BGP-HTTPS-TOPOLOGIA.pkt](BGP-HTTPS-TOPOLOGIA.pkt) | Equipos y cableado para resolver el ejercicio desde cero. |
| [BGP-HTTPS.pkt](BGP-HTTPS.pkt) | Ejemplo configurado para comprobar y comparar resultados. |

La versión `-TOPOLOGIA.pkt` conserva la distribución, los nombres visibles y los puertos utilizados. No contiene las IP, los protocolos ni los servicios personalizados del ejercicio; los dispositivos mantienen sus valores de fábrica. Los enlaces de los routers pueden aparecer desactivados hasta aplicar `no shutdown`.

Las tablas y comandos siguientes describen la versión configurada y permiten completar la topología base.

Trabaje en **Realtime** y espere a que los enlaces y protocolos converjan antes de probar. El primer ping puede perder paquetes por ARP o STP; repítalo para comprobar el estado estable.

## Subredes

| Segmento | Red/prefijo | Máscara | Hosts utilizables | Broadcast |
|---|---|---|---|---|
| Enlace entre AS | `10.0.0.0/30` | `255.255.255.252` | `10.0.0.1` – `10.0.0.2` | `10.0.0.3` |
| LAN AS 65001 | `192.168.10.0/24` | `255.255.255.0` | `192.168.10.1` – `192.168.10.254` | `192.168.10.255` |
| LAN AS 65002 | `192.168.20.0/24` | `255.255.255.0` | `192.168.20.1` – `192.168.20.254` | `192.168.20.255` |

## Direccionamiento de equipos

| Equipo | Interfaz | IP/prefijo | Gateway | DNS |
|---|---|---|---|---|
| R-AS65001 | GigabitEthernet0/0 | `10.0.0.1/30` | — | — |
| R-AS65001 | GigabitEthernet0/1 | `192.168.10.1/24` | — | — |
| R-AS65002 | GigabitEthernet0/0 | `10.0.0.2/30` | — | — |
| R-AS65002 | GigabitEthernet0/1 | `192.168.20.1/24` | — | — |
| SW-CLIENTES | Puertos de capa 2 | Sin IP de administración | — | — |
| SW-WEB | Puertos de capa 2 | Sin IP de administración | — | — |
| PC-BGP | FastEthernet0 | `192.168.10.10/24` | 192.168.10.1 | 192.168.20.10 |
| WEB-HTTPS | FastEthernet0 | `192.168.20.10/24` | 192.168.20.1 | — |

Las IP de clientes y servidores son estáticas. Se ingresan en **Desktop → IP Configuration**, junto con su máscara, gateway y DNS. Los switches conservan la VLAN 1 y conmutan en capa 2; no requieren IP para este ejemplo.

## Enlaces

| Origen | Destino | Cable |
|---|---|---|
| R-AS65001 GigabitEthernet0/0 | R-AS65002 GigabitEthernet0/0 | Cruzado |
| R-AS65001 GigabitEthernet0/1 | SW-CLIENTES GigabitEthernet0/1 | Directo |
| R-AS65002 GigabitEthernet0/1 | SW-WEB GigabitEthernet0/1 | Directo |
| SW-CLIENTES FastEthernet0/1 | PC-BGP FastEthernet0 | Directo |
| SW-WEB FastEthernet0/1 | WEB-HTTPS FastEthernet0 | Directo |

## Comandos de los routers

El archivo ya contiene la configuración. Estos bloques permiten reproducirla en una topología nueva con los mismos equipos y enlaces.

### R-AS65001

```cisco
enable
configure terminal
hostname R-AS65001
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
 exit
interface GigabitEthernet0/1
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit
router bgp 65001
neighbor 10.0.0.2 remote-as 65002
network 192.168.10.0 mask 255.255.255.0
no auto-summary
exit
end
write memory
```

### R-AS65002

```cisco
enable
configure terminal
hostname R-AS65002
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
 exit
interface GigabitEthernet0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
 exit
router bgp 65002
neighbor 10.0.0.1 remote-as 65001
network 192.168.20.0 mask 255.255.255.0
no auto-summary
exit
end
write memory
```

## Funcionamiento de eBGP

| Router | AS local | Vecino | AS remoto | Red anunciada |
|---|---:|---|---:|---|
| R-AS65001 | 65001 | `10.0.0.2` | 65002 | `192.168.10.0/24` |
| R-AS65002 | 65002 | `10.0.0.1` | 65001 | `192.168.20.0/24` |

`neighbor ... remote-as` establece la relación entre AS diferentes. `network ... mask` anuncia el prefijo cuando existe una ruta con esa red y máscara exactas en la tabla local. Las LAN están directamente conectadas. No se configuraron rutas estáticas ni rutas por defecto.

## DNS y servicio web

En **WEB-HTTPS → Services**:

- **DNS: On**, registro A `seguro.redes2.test` → `192.168.20.10`.
- **HTTP: On** y **HTTPS: On** para comparar ambos accesos.
- `index.html` identifica la semana 10 y el AS del servidor.

PC-BGP utiliza `192.168.20.10` como DNS. Al estar en otra LAN, la resolución DNS también depende de BGP y del gateway del cliente.

## Comandos de verificación

En ambos routers:

```cisco
show ip bgp summary
show ip bgp
show ip route
show ip protocols
```

En PC-BGP:

```text
ipconfig /all
ping 192.168.20.10
nslookup seguro.redes2.test
tracert 192.168.20.10
```

En **Web Browser**:

```text
http://seguro.redes2.test
https://seguro.redes2.test
```

| Comprobación realizada | Resultado observado |
|---|---|
| Vecindad BGP | Sesión establecida entre `10.0.0.1` y `10.0.0.2` |
| Ruta remota en R-AS65001 | `B 192.168.20.0/24` vía `10.0.0.2` |
| Ruta remota en R-AS65002 | `B 192.168.10.0/24` vía `10.0.0.1` |
| AS-PATH hacia la LAN del servidor | `65002` en R-AS65001 |
| Ping PC-BGP a WEB-HTTPS | 4/4 respuestas tras convergencia y ARP |
| DNS | `seguro.redes2.test` → `192.168.20.10` |
| HTTP y HTTPS | Ambos accesos devolvieron la página del servidor |

En `show ip bgp summary`, una sesión establecida presenta un número en `State/PfxRcd`, en lugar de `Idle` o `Active`. Confirme los prefijos reales con `show ip bgp` y `show ip route`; no dependa únicamente del contador mostrado por el simulador.

## iBGP, atributos y TLS

Esta topología ejecuta **eBGP**. iBGP conectaría vecinos del mismo AS y no está configurado. Se puede observar AS-PATH; no hay políticas LOCAL_PREF o MED ni caminos alternativos, por lo que no se demuestra selección entre varias salidas.

HTTP usa TCP 80 y HTTPS TCP 443. El servicio HTTPS del Server-PT es una simulación: que abra la página no demuestra una negociación real de TLS 1.3, una cadena de confianza pública ni la validez de un certificado real.

Para relacionarlo con la presentación, explique qué debe comprobar un cliente real:

1. Que el certificado corresponda al nombre solicitado.
2. Que esté dentro de su periodo de validez.
3. Que la firma y la cadena conduzcan a una autoridad de confianza.
4. Que se acuerden parámetros criptográficos adecuados.

DV valida control del dominio; OV/EV añaden validaciones de identidad. Esas categorías no significan por sí solas un cifrado más fuerte. La administración de certificados, autoridades y claves no se implementa en este `.pkt`.
