# Informe de configuración de DMZ con Cisco Packet Tracer


### 1. Objetivo del laboratorio

Configurar una Zona Desmilitarizada (DMZ) segura en Cisco Packet Tracer utilizando un router Cisco ISR como firewall central, aplicando:
- Direccionamiento IP,
- NAT estático y Listas de Control de Acceso (ACLs) para aislar el servidor web de la DMZ,
- Exponerlo de forma controlada hacia Internet,
- Bloquear el tráfico no autorizado entre la DMZ y la red interna (LAN).

---

### 2. Topología implementada

- **Cantidad de redes:** 3 (LAN Interna, DMZ, Red Externa/Internet)
- **Dispositivos usados:**
  - 1 router Cisco ISR (`Router_FW`) actuando como firewall central
  - 3 switches Cisco 2960 (`SW_Internal`, `SW_DMZ`, `SW_External`)
  - 1 PC en la red interna (`PC_Internal`)
  - 1 servidor web en la DMZ (`Server-PT Web_DMZ`)
  - 1 PC simulando un cliente externo/Internet (`PC_External`)

- **Función de cada zona:**
  - **LAN Interna:** Contiene los dispositivos internos de la organización. Debe estar protegida contra accesos no autorizados provenientes de la DMZ o de Internet.
  - **DMZ:** Zona donde se ubican los servicios que deben ser accesibles desde el exterior, en este caso un servidor web.
  - **Red Externa:** imula Internet y permite comprobar el acceso público al servidor publicado mediante NAT.

---

### 3. Plan de direccionamiento IP

| Dispositivo             | IP              | Máscara           | Gateway           |
|-------------------------|------------------|-------------------|-------------------|
| PC_Internal             | 192.168.1.10     | 255.255.255.0     | 192.168.1.1       |
| Server_DMZ              | 192.168.2.10     | 255.255.255.0     | 192.168.2.1       |
| PC_External             | 192.168.3.10     | 255.255.255.0     | 192.168.3.1       |
| Router_FW Gi0/0 (LAN)   | 192.168.1.1      | 255.255.255.0     | —                 |
| Router_FW Gi0/1 (DMZ)   | 192.168.2.1      | 255.255.255.0     | —                 |
| Router_FW Gi0/2 (Ext)   | 192.168.3.1      | 255.255.255.0     | —                 |

---

### 4. Configuración aplicada (resumen)

**Interfaces del router:**

```bash
Router> enable
Router# configure terminal
Router(config)# hostname Router_FW

Router_FW(config)# interface GigabitEthernet0/0
Router_FW(config-if)# ip address 192.168.1.1 255.255.255.0
Router_FW(config-if)# no shutdown
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/1
Router_FW(config-if)# ip address 192.168.2.1 255.255.255.0
Router_FW(config-if)# no shutdown
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/2
Router_FW(config-if)# ip address 192.168.3.1 255.255.255.0
Router_FW(config-if)# no shutdown
Router_FW(config-if)# exit

Router_FW(config)# end
Router_FW# write memory
```

**NAT estático**:

```bash
Router_FW(config)# interface GigabitEthernet0/1
Router_FW(config-if)# ip nat inside
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/2
Router_FW(config-if)# ip nat outside
Router_FW(config-if)# exit

Router_FW(config)# ip nat inside source static 192.168.2.10 192.168.3.1
```

Este mapeo uno a uno hace que el servidor (192.168.2.10) sea accesible desde la red externa a través de la IP pública 192.168.3.1.


**ACLs:**

```bash
! Permite únicamente tráfico HTTP desde Internet hacia el servidor DMZ
Router_FW(config)# access-list 100 permit tcp any host 192.168.3.1 eq 80
Router_FW(config)# interface GigabitEthernet0/2
Router_FW(config-if)# ip access-group 100 in
Router_FW(config-if)# exit

! Permite el tráfico de retorno de conexiones TCP ya establecidas desde el servidor DMZ hacia la LAN
Router_FW(config)# access-list 101 permit tcp host 192.168.2.10 any established

! Bloquea completamente el tráfico originado en la DMZ hacia la LAN interna
Router_FW(config)# access-list 101 deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
Router_FW(config)# access-list 101 permit ip any any
Router_FW(config)# interface GigabitEthernet0/1
Router_FW(config-if)# ip access-group 101 in
Router_FW(config-if)# exit

Router_FW(config)# end
Router_FW# write memory
```

La ACL 100 permite solo tráfico HTTP hacia el servidor y deniega implícitamente el resto (incluido ICMP/ping) al llegar por la interfaz externa. La ACL 101 deniega explícitamente cualquier paquete que vaya de la red DMZ hacia la LAN, con una excepción previa: permite el tráfico de retorno de conexiones TCP ya establecidas desde el servidor (192.168.2.10), sin la cual PC_Internal no podía completar su acceso web al servidor DMZ, ya que la respuesta del servidor quedaba bloqueada por la regla de denegación.

---

### 5. Verificaciones realizadas

**Conectividad básica (antes de aplicar ACLs):**
- `ping` desde `PC_Internal` a su gateway (192.168.1.1): ✅ Replies exitosas
- `ping` desde `Server-PT Web_DMZ` a su gateway (192.168.2.1): ✅ Replies exitosas
- `ping` desde `PC_External` a su gateway (192.168.3.1): ✅ Replies exitosas

**Prueba de acceso web inicial (antes de ACLs):**
- Acceso desde `PC_External` a `http://192.168.3.1` (IP pública vía NAT): ✅ Carga la página del servidor
- Acceso desde `PC_Internal` a `http://192.168.2.10` (IP directa DMZ): ✅ Carga la página del servidor

**Verificación final de seguridad y funcionalidad (con ACLs aplicadas):**
- Acceso web desde `PC_External` al servidor DMZ (192.168.3.1): ✅ La página carga correctamente
- `ping` desde `PC_External` a 192.168.3.1: ✅ Bloqueado (Request timed out)
- Acceso web desde `PC_Internal` al servidor DMZ (192.168.2.10): ✅ La página carga correctamente
- `ping` desde `Server-PT Web_DMZ` a `PC_Internal` (192.168.1.10): ✅ Bloqueado (Request timed out)

---

### 6. Conclusiones y recomendaciones

Este laboratorio permitió aplicar de forma práctica los conceptos de segmentación de red mediante una DMZ, reforzando la diferencia entre exponer un servicio de forma controlada (NAT + ACL permitiendo solo el puerto necesario) y aislar completamente zonas de distinta confianza (ACL de denegación total DMZ→LAN). La DMZ es una capa intermedia: permite que Internet llegue al servidor web sin que ese mismo servidor, en caso de verse comprometido, pueda usarse como punto de pivote hacia la red interna.

Como recomendación, es fundamental verificar la conectividad básica (ping a los gateways) *antes* de aplicar cualquier ACL, ya que un error de IP o máscara es mucho más difícil de diagnosticar una vez que las reglas de firewall ya están activas y empiezan a bloquear tráfico. También es importante recordar que las ACLs extendidas tienen una denegación implícita al final, por lo que hay que ser explícito sobre qué tráfico se desea permitir (por ejemplo, agregar `permit ip any any` en la ACL 101 para no bloquear accidentalmente tráfico legítimo de la DMZ hacia Internet).

Un hallazgo importante durante la implementación fue que, al aplicar la ACL que deniega el tráfico DMZ→LAN, PC_Internal dejó de poder acceder al servidor web de la DMZ. Esto ocurrió porque una ACL de este tipo no distingue entre una conexión nueva iniciada por el servidor y el tráfico de retorno de una conexión que PC_Internal inició. La solución fue agregar access-list 101 permit tcp host 192.168.2.10 any established antes de la regla de denegación, lo que permite únicamente los paquetes de respuesta de conexiones ya iniciadas desde la LAN, sin abrir la puerta a que el servidor inicie tráfico nuevo hacia la red interna. Esto refuerza la importancia de probar la funcionalidad completa después de aplicar cada ACL, y no asumir que una regla de seguridad no tendrá efectos colaterales sobre el tráfico legítimo.

---

### 7. Capturas de evidencia (Anexo)

> - Topología general de la red en Packet Tracer
> - `show running-config` del `Router_FW`
> - Resultado del `ping` desde `PC_Internal`, `Server-PT Web_DMZ` y `PC_External` a sus respectivos gateways
> - Acceso web exitoso desde `PC_External` y `PC_Internal` al servidor DMZ
> - `ping` fallido desde `PC_External` a 192.168.3.1 (bloqueado por ACL)
> - `ping` fallido desde `Server-PT Web_DMZ` a `PC_Internal` (bloqueado por ACL)