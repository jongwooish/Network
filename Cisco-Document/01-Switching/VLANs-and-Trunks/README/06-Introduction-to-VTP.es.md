# Introduction to VLAN Trunking Protocol (VTP)

[한국어](../06-Introduction-to-VTP.md) | [English](06-Introduction-to-VTP.en.md) | [日本語](06-Introduction-to-VTP.ja.md) | [简体中文](06-Introduction-to-VTP.zh-CN.md) | **Español** | [Français](06-Introduction-to-VTP.fr.md)

Si tienes 20 switches y 50 VLAN, crear cada VLAN a mano en cada switch lleva demasiado tiempo. Con **VTP (VLAN Trunking Protocol)**, creas las VLAN en un switch y los demás reciben esa información y **se sincronizan automáticamente**.

- Protocolo propietario de Cisco
- La información de VTP viaja **solo por enlaces trunk**
- Solo se sincronizan entre sí los switches con el mismo **nombre de dominio VTP**

---

## 1. Cómo funciona VTP

![VTP domain](../images/vtp-domain.svg)

1. Añade / modifica / elimina VLAN en el **VTP Server**.
2. Cada cambio aumenta en 1 el número de **Configuration Revision**.
3. La información más reciente (advertisement) se envía a todos los VTP Client.
4. Un VTP Client se sincroniza cuando recibe información con una **revisión mayor que la suya**.

> Un VTP Server también se sincroniza (hace también de cliente). Por eso puedes tener varios VTP Server y cambiar las VLAN en cualquiera de ellos.

---

## 2. Modos de VTP

### VTP Transparent

![VTP transparent](../images/vtp-transparent.svg)

Un switch VTP Transparent **solo reenvía** los advertisements a otros switches y **no se sincroniza**. A cambio, puede crear VLAN **de forma local**.

Cuando creas la VLAN 20 en el VTP Server:
1. Se crea la VLAN 20 en el VTP Server
2. Aumenta el número de revisión
3. El VTP Server envía el advertisement al switch Transparent
4. El Transparent no se sincroniza, pero **reenvía** el advertisement al Client
5. El Client se sincroniza con la información más reciente

### Comparación de modos

| | **VTP Server** | **VTP Client** | **VTP Transparent** |
| --- | --- | --- | --- |
| Crear / modificar / eliminar VLAN | ✅ | ❌ | Solo en local |
| Se sincroniza | ✅ | ✅ | ❌ |
| Reenvía advertisements | ✅ | ✅ | ✅ |
| Dónde se guarda la información de VLAN | `vlan.dat` | `vlan.dat` | **running-config** (+ `vlan.dat`) |

> También existe el **modo `off`**, que desactiva VTP por completo. Se parece a Transparent, pero **tampoco reenvía** advertisements (VTP versión 3, o IOS que lo admitan).

---

## 3. El peligro de VTP: el número de revisión

VTP es cómodo, pero tiene un riesgo serio. **Un VTP Server también se sincroniza como un Client**, y todos los switches siguen la información con el **número de revisión más alto**.

Puede pasar esto:

1. Sacas un switch VTP Client de la red de producción y lo llevas a un laboratorio
2. Lo cambias a modo VTP Server
3. Creas y modificas VLAN mientras haces pruebas → la **revisión aumenta** con cada cambio
4. Al terminar, **eliminas todas las VLAN**
5. Lo vuelves a poner en modo VTP Client
6. **Reconectas** ese switch a la red de producción

Como la revisión de este switch es **mayor** que la de la red de producción, los demás switches se sincronizan con él → **desaparecen todas las VLAN de producción.** Los puertos de esas VLAN ni siquiera vuelven a la VLAN 1: se quedan en VLAN que ya no existen y **el tráfico se corta.**

> **Prevención**
> - Antes de conectar un switch que se usó en otro sitio, **pon su revisión a 0**: cambia temporalmente el nombre de dominio VTP y vuelve a ponerlo, o pasa a modo Transparent y vuelve.
> - Configura una contraseña de dominio con `vtp password`.
> - Con **VTP versión 3**, solo el switch designado como **Primary Server** puede cambiar la información de VLAN, lo que evita este accidente.
> - Muchas redes que no necesitan VTP funcionan directamente en modo **Transparent** (u `off`).

---

## 4. VTP pruning

![VTP pruning](../images/vtp-pruning.svg)

Hay PC en las VLAN 10, 20 y 30, y los enlaces entre switches son **trunks 802.1Q** que transportan todas las VLAN.
Cuando un PC de la VLAN 10 envía un **broadcast**, los switches lo inundan y los trunks transportan todas las VLAN, así que **llega a todos los switches.**

Pero el switch del centro **no tiene ningún PC de la VLAN 10** (solo de la VLAN 20). Ese broadcast es **ancho de banda desperdiciado**.

Con **VTP pruning** activado, el tráfico de una VLAN no se envía por los trunks hacia switches que no tienen equipos en esa VLAN.

```
Switch(config)#vtp pruning
```

- Al activarlo en un Server, se aplica a todo el dominio.
- Según el modelo de switch, viene activado o desactivado por defecto.
- La VLAN 1 y las VLAN 1002 ~ 1005 nunca se podan.

---

## 5. Práctica de configuración de VTP

![VTP lab topology](../images/vtp-lab-topology.svg)

Tres switches (SW1, SW2, SW3) están conectados con trunks. Se han borrado la base de datos de VLAN y la startup-config de todos ellos.

### 5-1. Comprobar el estado por defecto

```
SW1#show vtp status
VTP Version capable             : 1 to 3
VTP version running             : 1
VTP Domain Name                 :
VTP Pruning Mode                : Disabled
VTP Traps Generation            : Disabled
Configuration last modified by 0.0.0.0 at 0-0-00 00:00:00

Feature VLAN:
--------------
VTP Operating Mode                : Server
Maximum VLANs supported locally   : 1005
Number of existing VLANs          : 5
Configuration Revision            : 0
```

SW2 y SW3 muestran lo mismo.

| Campo | Significado |
| --- | --- |
| `Configuration Revision : 0` | Aumenta cada vez que se añaden o eliminan VLAN. Sigue en 0 porque no se ha cambiado nada |
| `VTP Operating Mode : Server` | El modo por defecto es **Server** |
| `VTP Pruning Mode : Disabled` | El pruning está desactivado |
| `VTP version running : 1` | Admite las versiones 1–3, pero ahora usa la versión 1 |
| `VTP Domain Name :` (vacío) | Todavía no hay nombre de dominio |

### 5-2. Crear una VLAN en SW1

```
SW1(config)#vlan 10
SW1(config-vlan)#name Printers
```

```
SW1#show vlan
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi0/0, Gi0/1, Gi0/2, Gi0/3
                                                Gi1/0, Gi1/1, Gi1/2, Gi1/3
10   Printers                         active
```

```
SW1#show vtp status | include Revision
Configuration Revision            : 1
```

La revisión de SW1 ha subido a **1**. Pero SW2 y SW3 siguen en **0** y no tienen la VLAN 10.
→ Porque no se ha configurado ningún **nombre de dominio VTP**.

### 5-3. Configurar el dominio VTP

Activa el debug en SW2 y SW3 para verlo en tiempo real.

```
SW2#debug sw-vlan vtp events
vtp events debugging is on
```

Configura el nombre de dominio en SW1.

```
SW1(config)#vtp domain JJONG
Changing VTP domain name from NULL to JJONG
```

Salida del debug en SW3:

```
VTP LOG RUNTIME: Summary packet received in NULL domain state
VTP LOG RUNTIME: Summary packet received, domain = JJONG, rev = 1, followers = 1, length 77, trunk Gi0/0
VTP LOG RUNTIME: Transitioning from NULL to JJONG domain
VTP LOG RUNTIME: Summary packet rev 1 greater than domain JJONG rev 0
```

- Un switch **sin nombre de dominio (NULL)** **adopta** el nombre de dominio del paquete VTP recibido (`JJONG`). (No lo cambia si ya tiene un nombre de dominio.)
- La revisión del paquete recibido (**1**) es mayor que la suya (**0**), así que **se sincroniza.**

El debug genera mucha salida, así que desactívalo siempre.

```
SW2#no debug all
All possible debugging has been turned off
```

```
SW2#show vtp status | include Domain|Revision
VTP Domain Name                 : JJONG
Configuration Revision          : 1
```

```
SW2#show vlan
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi0/1, Gi0/3, Gi1/0, Gi1/1
                                                Gi1/2, Gi1/3
10   Printers                         active
```

SW2 y SW3 han **aprendido la VLAN 10** mediante VTP.

### 5-4. Crear VLAN en varios Server

Ahora los tres switches son Server, así que se pueden crear VLAN en cualquiera de ellos.

```
SW2(config)#vlan 20
SW2(config-vlan)#name Servers
```
```
SW3(config)#vlan 30
SW3(config-vlan)#name Management
```

```
SW1#show vlan
VLAN Name                             Status
---- -------------------------------- ---------
1    default                          active
10   Printers                         active
20   Servers                          active
30   Management                       active
```

SW2 y SW3 también tienen las VLAN 10, 20 y 30, y la revisión es **3** en los tres switches. Sube 1 con cada VLAN nueva.

### 5-5. Modo VTP Client

```
SW2(config)#vtp mode client
Setting device to VTP Client mode for VLANS.
```

```
SW2#show vtp status | include Operating
VTP Operating Mode                : Client
```

Ahora SW1 y SW3 son Server y SW2 es Client. Para esta prueba, **desconecta el enlace SW1 ↔ SW3** para que no haya conexión directa entre ellos. (SW1 — SW2 — SW3)

```
SW1(config)#vlan 40
SW1(config-vlan)#name Engineering
```

```
SW2#show vlan
VLAN Name                             Status
---- -------------------------------- ---------
1    default                          active
10   Printers                         active
20   Servers                          active
30   Management                       active
40   Engineering                      active
```

SW3 también aprende la VLAN 40, por el camino SW1 → **SW2 (Client)** → SW3.
→ **Un Client se sincroniza y además reenvía los advertisements a otros switches.**

En un Client no se pueden crear VLAN.

```
SW2(config)#vlan 50
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

### 5-6. Modo VTP Transparent

Con el enlace SW1 ↔ SW3 todavía desconectado, cambia SW2 a Transparent.

```
SW2(config)#vtp mode transparent
Setting device to VTP Transparent mode for VLANS.
```

Crea la VLAN 50 en SW1.

```
SW1(config)#vlan 50
SW1(config-vlan)#name Research
```

| Switch | VLAN 50 | Por qué |
| --- | --- | --- |
| SW1 (Server) | ✅ Presente | Se creó ahí |
| SW2 (Transparent) | ❌ Ausente | Transparent **no se sincroniza** |
| SW3 (Server) | ✅ Presente | SW2 **reenvió** el advertisement |

Ahora crea la VLAN 60 en SW2 (Transparent).

```
SW2(config)#vlan 60
SW2(config-vlan)#name Cameras
```

```
SW2#show vlan
VLAN Name                             Status
---- -------------------------------- ---------
1    default                          active
10   Printers                         active
20   Servers                          active
30   Management                       active
40   Engineering                      active
60   Cameras                          active
```

Un switch Transparent puede crear VLAN **en local**. Pero la VLAN 60 **no aparece** en SW1 ni en SW3, porque las VLAN de un switch Transparent son solo locales y no se anuncian.

### 5-7. Transparent guarda las VLAN en la running-config

```
SW2#show running-config
...
vlan 10
 name Printers
!
vlan 20
 name Servers
!
vlan 30
 name Management
!
vlan 40
 name Engineering
!
vlan 60
 name Cameras
!
```

| Modo | Dónde se guarda la información de VLAN |
| --- | --- |
| Server / Client | `flash:vlan.dat` (no aparece en la running-config) |
| **Transparent** | También en la **running-config** → guardar con `copy run start` |

---

## 6. Resumen de comandos

| Comando | Descripción |
| --- | --- |
| `vtp domain <nombre>` | Configura el nombre de dominio VTP (distingue mayúsculas) |
| `vtp mode {server \| client \| transparent \| off}` | Cambia el modo VTP |
| `vtp version {1 \| 2 \| 3}` | Cambia la versión de VTP |
| `vtp password <contraseña>` | Configura la contraseña del dominio |
| `vtp pruning` | Activa el VTP pruning |
| `show vtp status` | Muestra versión, dominio, modo y revisión |
| `show vtp password` | Muestra la contraseña configurada |
| `debug sw-vlan vtp events` | Muestra los eventos VTP en tiempo real (se desactiva con `no debug all`) |

---

## 7. Ideas clave

- VTP **sincroniza automáticamente** las VLAN creadas en un switch con los switches del mismo dominio. Solo funciona si los **nombres de dominio coinciden** y viaja únicamente por **trunks**.
- **Server**: crea VLAN + se sincroniza + reenvía / **Client**: se sincroniza + reenvía / **Transparent**: solo reenvía, sus VLAN son locales
- Todos los switches siguen la información con la **revisión más alta**. Pon a **0 la revisión** de un switch usado antes de conectarlo, o podrías borrar todas las VLAN.
- El **VTP pruning** ahorra ancho de banda bloqueando el tráfico de VLAN en los trunks donde no hace falta.
- Transparent guarda las VLAN en la **running-config**; Server/Client las guardan en **vlan.dat**.
