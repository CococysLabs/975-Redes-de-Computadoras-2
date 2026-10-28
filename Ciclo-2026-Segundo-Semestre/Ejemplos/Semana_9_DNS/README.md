# Semana 9 - DNS y servicios de nombres

## Objetivo

Configurar resolución de nombres con un registro A y un alias CNAME, comparar el acceso por nombre y por IP y observar una consulta DNS.

## Archivo y uso

Use **Cisco Packet Tracer 8.2.2 o una versión compatible**.

| Archivo | Uso |
|---|---|
| [DNS-Servicios-Web-TOPOLOGIA.pkt](DNS-Servicios-Web-TOPOLOGIA.pkt) | Equipos y cableado para resolver el ejercicio desde cero. |
| [DNS-Servicios-Web.pkt](DNS-Servicios-Web.pkt) | Ejemplo configurado para comprobar y comparar resultados. |

La versión `-TOPOLOGIA.pkt` conserva la distribución, los nombres visibles y los puertos utilizados. No contiene las IP, los protocolos ni los servicios personalizados del ejercicio; los dispositivos mantienen sus valores de fábrica. Los enlaces de los routers pueden aparecer desactivados hasta aplicar `no shutdown`.

Las tablas y comandos siguientes describen la versión configurada y permiten completar la topología base.

Trabaje en **Realtime** y espere a que los enlaces y protocolos converjan antes de probar. El primer ping puede perder paquetes por ARP o STP; repítalo para comprobar el estado estable.

## Subredes

| Segmento | Red/prefijo | Máscara | Hosts utilizables | Broadcast |
|---|---|---|---|---|
| LAN DNS/HTTP | `192.168.9.0/24` | `255.255.255.0` | `192.168.9.1` – `192.168.9.254` | `192.168.9.255` |

## Direccionamiento de equipos

| Equipo | Interfaz | IP/prefijo | Gateway | DNS |
|---|---|---|---|---|
| R-DNS | GigabitEthernet0/0 | `192.168.9.1/24` | — | — |
| SW-DNS | Puertos de capa 2 | Sin IP de administración | — | — |
| DNS-WEB | FastEthernet0 | `192.168.9.10/24` | 192.168.9.1 | — |
| PC-DNS1 | FastEthernet0 | `192.168.9.20/24` | 192.168.9.1 | 192.168.9.10 |
| PC-DNS2 | FastEthernet0 | `192.168.9.21/24` | 192.168.9.1 | 192.168.9.10 |

Las IP de clientes y servidores son estáticas. Se ingresan en **Desktop → IP Configuration**, junto con su máscara, gateway y DNS. Los switches conservan la VLAN 1 y conmutan en capa 2; no requieren IP para este ejemplo.

## Enlaces

| Origen | Destino | Cable |
|---|---|---|
| R-DNS GigabitEthernet0/0 | SW-DNS GigabitEthernet0/1 | Directo |
| DNS-WEB FastEthernet0 | SW-DNS FastEthernet0/1 | Directo |
| PC-DNS1 FastEthernet0 | SW-DNS FastEthernet0/2 | Directo |
| PC-DNS2 FastEthernet0 | SW-DNS FastEthernet0/3 | Directo |

## Comandos de los routers

El archivo ya contiene la configuración. Estos bloques permiten reproducirla en una topología nueva con los mismos equipos y enlaces.

### R-DNS

```cisco
enable
configure terminal
hostname R-DNS
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 192.168.9.1 255.255.255.0
 no shutdown
 exit
end
write memory
```

## Servicios DNS y HTTP

En **DNS-WEB → Services → DNS**, deje **DNS Service: On** y configure:

| Nombre | Tipo | Valor |
|---|---|---|
| `www.redes2.test` | A | `192.168.9.10` |
| `aula.redes2.test` | CNAME | `www.redes2.test` |

En **Services → HTTP**, deje **HTTP: On**. La página `index.html` identifica la semana 9. HTTPS no se utiliza en este ejercicio.

En PC-DNS1 y PC-DNS2, el servidor DNS debe ser `192.168.9.10`. El router actúa como gateway de la LAN, pero no es necesario para la comunicación entre estos clientes y el servidor, porque pertenecen a la misma subred. No hace falta Internet para resolver los nombres internos.

## Comandos de verificación

En **Desktop → Command Prompt** de los clientes:

```text
ipconfig /all
ping 192.168.9.10
nslookup www.redes2.test
nslookup aula.redes2.test
ping www.redes2.test
```

En **Desktop → Web Browser**, compare:

```text
http://192.168.9.10
http://www.redes2.test
http://aula.redes2.test
```

| Comprobación | Resultado |
|---|---|
| Registro A | `www.redes2.test` resuelve a `192.168.9.10` |
| Alias CNAME | Resuelve mediante `www.redes2.test` a la misma IP |
| Ping por IP | 4/4 respuestas después de estabilizar enlaces |
| Navegación por IP y por nombre A | Muestra la página de la semana 9 |

## Consulta y respuesta en Simulation

1. Cambie de **Realtime** a **Simulation**.
2. En **Edit Filters**, muestre DNS y ARP para reducir el ruido de otros protocolos.
3. Ejecute `nslookup aula.redes2.test` desde PC-DNS1.
4. Use **Capture/Forward** hasta observar la respuesta en el cliente.
5. Abra los detalles de los eventos y compare el nombre consultado, la dirección del servidor y el contenido de la respuesta.
6. Vuelva a **Realtime** al terminar.

Se verificó la consulta desde PC-DNS1, su paso por SW-DNS, la respuesta de DNS-WEB y la recepción en el cliente. Los descartes de tramas de control o broadcasts en equipos que no son el destinatario no equivalen a un fallo de resolución.

## Diagnóstico y alcance

- Si falla el ping por IP, revise cable, IP, máscara y estado del puerto.
- Si funciona por IP pero no por nombre, revise el DNS del cliente, el servicio y los registros.
- Si resuelve el nombre pero no carga la página, revise HTTP y el URL.
- Un A contiene una IPv4; un CNAME apunta a otro nombre, no directamente a una IP.
- AAAA, MX, NS y TXT cumplen otras funciones. Este archivo demuestra A y CNAME; no implementa una jerarquía raíz/TLD, transferencias de zona ni DNSSEC.
- DNS puede usar UDP o TCP en el puerto 53. La simulación permite observar el tráfico que genera esta consulta concreta.
