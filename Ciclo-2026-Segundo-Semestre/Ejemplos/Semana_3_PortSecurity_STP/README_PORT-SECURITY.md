# Clase 2

Clase 2 port-security.pkt

## Switch0

```bash
interface FastEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security violation protect
 switchport port-security mac-address 000A.F3CC.4965
!
interface FastEthernet0/2
 switchport mode access
 switchport port-security
 switchport port-security violation restrict
 switchport port-security mac-address 0001.63AA.3CC8
!
interface FastEthernet0/3
 switchport mode access
 switchport port-security
 switchport port-security violation shutdown
 switchport port-security mac-address 00D0.58A1.AA01
```
