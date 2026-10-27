# Configuración de STP para la topología con VLAN 10 y VLAN 20

## 1. Objetivo de la configuración

Esta topología tiene tres switches conectados en forma de triángulo. Esa conexión proporciona redundancia, porque si uno de los enlaces falla, todavía existe otro camino para transportar el tráfico.

Sin embargo, si los tres enlaces permanecieran reenviando tramas al mismo tiempo, se produciría un **bucle de capa 2**. Un bucle puede causar:

- Tormentas de broadcast.
- Duplicación de tramas.
- Inestabilidad en las tablas MAC.
- Saturación de los enlaces.
- Pérdida parcial o total de comunicación.

Para evitarlo se utilizará **Rapid PVST+**, que ejecuta una instancia de Spanning Tree por cada VLAN y converge más rápido que el STP tradicional.

La estrategia será la siguiente:

- El switch superior será el **Root Bridge principal** para las VLAN 10 y 20.
- El switch inferior izquierdo será el **Root Bridge secundario** para las VLAN 10 y 20.
- El switch inferior derecho quedará como switch de acceso.
- STP bloqueará automáticamente uno de los enlaces redundantes del triángulo.
- Si un enlace activo falla, STP habilitará el enlace alternativo.

---

## 2. Identificación de los switches

Para esta guía se usarán los siguientes nombres:

| Nombre en la guía | Ubicación en la topología | Función |
|---|---|---|
| `SW-CORE` | Switch superior conectado al router | Root Bridge principal |
| `SW-ACCESS-1` | Switch inferior izquierdo | Root Bridge secundario |
| `SW-ACCESS-2` | Switch inferior derecho | Switch de acceso |

### Enlaces entre switches

Según la topología:

- `SW-CORE Fa0/20` conecta con `SW-ACCESS-1 Fa0/20`.
- `SW-CORE Fa0/21` conecta con `SW-ACCESS-2 Fa0/20`.
- `SW-ACCESS-1 Fa0/21` conecta con `SW-ACCESS-2 Fa0/21`.

> Los puertos entre switches deben funcionar como enlaces troncales. En esta guía se incluyen únicamente los comandos relacionados directamente con STP; se asume que las VLAN 10 y 20 y los enlaces trunk ya fueron configurados.

---

# 3. Configuración de STP en `SW-CORE`

El switch superior será el Root Bridge principal para las VLAN 10 y 20.

## 3.1 Ingresar al modo de configuración

```bash
enable
configure terminal
```

## 3.2 Cambiar el nombre del switch

```bash
hostname SW-CORE
```

Este comando permite identificar fácilmente el switch durante la configuración y las verificaciones.

## 3.3 Activar Rapid PVST+

```bash
spanning-tree mode rapid-pvst
```

### ¿Qué hace este comando?

Activa Rapid PVST+, una versión mejorada de Spanning Tree que:

- Mantiene una instancia de STP por cada VLAN.
- Detecta cambios de topología con mayor rapidez.
- Reduce el tiempo de recuperación cuando falla un enlace.
- Permite definir diferentes raíces para diferentes VLAN si fuera necesario.

## 3.4 Configurar el switch como Root Bridge principal

```bash
spanning-tree vlan 10,20 root primary
```

### ¿Qué hace este comando?

Reduce automáticamente la prioridad STP del switch para que tenga mayor posibilidad de convertirse en Root Bridge de las VLAN 10 y 20.

El Root Bridge es el punto de referencia que STP utiliza para calcular el camino más corto y decidir qué enlaces deben reenviar o bloquear tráfico.

Al estar conectado directamente al router, conviene que el switch superior sea la raíz, porque así el tráfico de ambas VLAN sigue un camino directo hacia la puerta de enlace.

## 3.5 Proteger el puerto conectado al router

El puerto `GigabitEthernet0/1` está conectado al router. Como el router no participa en STP, se puede activar PortFast en ese puerto.

```bash
interface gigabitEthernet0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
```

### ¿Qué hace PortFast?

Permite que el puerto pase inmediatamente al estado de reenvío, sin esperar la transición normal de STP.

### ¿Qué hace BPDU Guard?

Deshabilita el puerto si recibe una BPDU. Esto ayuda a evitar que alguien conecte accidentalmente otro switch en un puerto que debería estar conectado únicamente al router.

> No se debe activar PortFast ni BPDU Guard en `Fa0/20` y `Fa0/21`, porque esos puertos conectan con otros switches y necesitan participar normalmente en STP.

## 3.6 Guardar la configuración

```bash
end
copy running-config startup-config
```

---

# 4. Configuración de STP en `SW-ACCESS-1`

El switch inferior izquierdo será el Root Bridge secundario. Si el switch principal falla, este switch podrá asumir la función de raíz.

## 4.1 Ingresar al modo de configuración

```bash
enable
configure terminal
```

## 4.2 Cambiar el nombre del switch

```bash
hostname SW-ACCESS-1
```

## 4.3 Activar Rapid PVST+

```bash
spanning-tree mode rapid-pvst
```

Todos los switches de la topología deben usar el mismo modo de STP para mantener un funcionamiento consistente.

## 4.4 Configurar el switch como Root Bridge secundario

```bash
spanning-tree vlan 10,20 root secondary
```

### ¿Qué hace este comando?

Configura una prioridad menor que la prioridad predeterminada, pero mayor que la del Root Bridge principal.

Por ello:

- Mientras `SW-CORE` esté funcionando, seguirá siendo la raíz.
- Si `SW-CORE` deja de participar en STP, `SW-ACCESS-1` podrá convertirse en la nueva raíz.

## 4.5 Activar PortFast y BPDU Guard en los puertos de computadoras

De acuerdo con la imagen, los equipos finales están conectados en puertos como `Fa0/1`, `Fa0/2` y `Fa0/10`.

```bash
interface range fastEthernet0/1 - 2
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit

interface fastEthernet0/10
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
```

### ¿Por qué se configura en los puertos de acceso?

Los puertos conectados a computadoras no necesitan participar en la elección del Root Bridge. PortFast permite que la computadora tenga conectividad más rápido y BPDU Guard protege la red si alguien conecta un switch no autorizado.

> Verifique en Packet Tracer los puertos exactos de las computadoras. Si un equipo está conectado a otro puerto, aplique los mismos comandos sobre ese puerto.

## 4.6 No activar PortFast en los enlaces entre switches

Los puertos siguientes deben participar normalmente en STP:

- `Fa0/20`, conectado a `SW-CORE`.
- `Fa0/21`, conectado a `SW-ACCESS-2`.

No se debe ejecutar PortFast sobre ellos.

## 4.7 Guardar la configuración

```bash
end
copy running-config startup-config
```

---

# 5. Configuración de STP en `SW-ACCESS-2`

El switch inferior derecho funcionará como switch de acceso. STP decidirá cuál de sus enlaces redundantes debe mantenerse en reenvío y cuál debe quedar bloqueado.

## 5.1 Ingresar al modo de configuración

```bash
enable
configure terminal
```

## 5.2 Cambiar el nombre del switch

```bash
hostname SW-ACCESS-2
```

## 5.3 Activar Rapid PVST+

```bash
spanning-tree mode rapid-pvst
```

## 5.4 Mantener la prioridad predeterminada

No es necesario configurar este switch como Root Bridge, porque `SW-CORE` será la raíz principal y `SW-ACCESS-1` será la secundaria.

Opcionalmente, se puede dejar explícita la prioridad predeterminada:

```bash
spanning-tree vlan 10,20 priority 32768
```

### ¿Qué hace este comando?

Mantiene una prioridad superior a la de los switches principal y secundario, evitando que este switch se convierta accidentalmente en Root Bridge.

## 5.5 Activar PortFast y BPDU Guard en los puertos de computadoras

Según la topología, las computadoras están conectadas en puertos como `Fa0/1` y `Fa0/10`.

```bash
interface fastEthernet0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit

interface fastEthernet0/10
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
```

## 5.6 No activar PortFast en los enlaces entre switches

Los siguientes puertos deben participar normalmente en STP:

- `Fa0/20`, conectado a `SW-CORE`.
- `Fa0/21`, conectado a `SW-ACCESS-1`.

## 5.7 Guardar la configuración

```bash
end
copy running-config startup-config
```

---

# 6. Comportamiento esperado de STP

Después de aplicar la configuración:

1. `SW-CORE` debe aparecer como Root Bridge para las VLAN 10 y 20.
2. Los enlaces de `SW-CORE` hacia los dos switches inferiores deben quedar en estado de reenvío.
3. El enlace horizontal entre los dos switches inferiores es redundante.
4. STP bloqueará uno de los extremos del enlace horizontal para evitar el bucle.
5. Si falla uno de los enlaces hacia `SW-CORE`, el puerto bloqueado podrá pasar a reenvío y restablecer la comunicación.

El puerto exacto que quede bloqueado puede variar según:

- La prioridad de los switches.
- El costo de los enlaces.
- El Port ID utilizado como criterio de desempate.
- La dirección MAC incluida en el Bridge ID.

---

# 7. Comandos de verificación

## 7.1 Verificar STP para todas las VLAN

Ejecutar en cada switch:

```bash
show spanning-tree
```

Este comando muestra:

- El Root Bridge actual.
- La prioridad y dirección MAC del switch raíz.
- El costo para llegar a la raíz.
- El rol de cada puerto.
- El estado de cada puerto.

## 7.2 Verificar STP para la VLAN 10

```bash
show spanning-tree vlan 10
```

En `SW-CORE` debe mostrarse un mensaje similar a:

```text
This bridge is the root
```

## 7.3 Verificar STP para la VLAN 20

```bash
show spanning-tree vlan 20
```

En `SW-CORE` también debe aparecer como Root Bridge.

## 7.4 Verificar la configuración aplicada

```bash
show running-config | include spanning-tree
```

## 7.5 Verificar puertos PortFast

```bash
show spanning-tree summary
```

## 7.6 Verificar puertos bloqueados

```bash
show spanning-tree blockedports
```

En condiciones normales debe existir al menos un puerto bloqueado dentro del triángulo, porque STP debe eliminar lógicamente el camino redundante.

---

# 8. Roles y estados que pueden observarse

## Roles de puerto

- **Root Port:** mejor puerto de un switch para llegar al Root Bridge.
- **Designated Port:** puerto seleccionado para reenviar tráfico en un segmento.
- **Alternate Port:** enlace alternativo que permanece listo para reemplazar al enlace principal.

## Estados principales en Rapid PVST+

- **Discarding:** no reenvía tramas de usuario. Puede ser un puerto bloqueado o en transición.
- **Learning:** aprende direcciones MAC, pero todavía no reenvía tráfico de usuario.
- **Forwarding:** aprende direcciones MAC y reenvía tráfico normalmente.

---

# 9. Prueba de redundancia

Para comprobar que STP funciona correctamente:

1. Verificar conectividad entre equipos de la misma VLAN.
2. Ejecutar `show spanning-tree vlan 10` y `show spanning-tree vlan 20`.
3. Identificar el puerto que aparece como `Alternate` o `Discarding`.
4. Desconectar temporalmente uno de los enlaces activos hacia `SW-CORE`.
5. Esperar unos segundos.
6. Verificar que el puerto alternativo cambie a `Forwarding`.
7. Comprobar nuevamente la comunicación con `ping`.
8. Reconectar el enlace y observar cómo STP vuelve a calcular la topología.

Rapid PVST+ debe realizar esta recuperación más rápido que el STP tradicional.

---

# 10. Resumen de comandos por switch

## `SW-CORE`

```bash
enable
configure terminal
hostname SW-CORE
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20 root primary
interface gigabitEthernet0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
end
copy running-config startup-config
```

## `SW-ACCESS-1`

```bash
enable
configure terminal
hostname SW-ACCESS-1
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20 root secondary
interface range fastEthernet0/1 - 2
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
interface fastEthernet0/10
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
end
copy running-config startup-config
```

## `SW-ACCESS-2`

```bash
enable
configure terminal
hostname SW-ACCESS-2
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20 priority 32768
interface fastEthernet0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
interface fastEthernet0/10
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
end
copy running-config startup-config
```

---

# 11. Recomendaciones importantes

- No activar PortFast en enlaces entre switches.
- No activar BPDU Guard en enlaces troncales entre switches.
- Mantener el mismo modo STP en los tres switches.
- Verificar que las VLAN 10 y 20 existan en todos los switches.
- Verificar que los enlaces entre switches transporten las VLAN 10 y 20.
- Guardar la configuración después de comprobar que STP funciona correctamente.
- Si un puerto entra en estado `err-disabled` por BPDU Guard, revisar primero qué dispositivo fue conectado antes de recuperarlo.

Para recuperar un puerto después de corregir la causa:

```bash
configure terminal
interface fastEthernet0/X
 shutdown
 no shutdown
end
```
