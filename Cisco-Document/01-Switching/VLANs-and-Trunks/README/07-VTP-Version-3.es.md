# VTP Version 3

[한국어](../07-VTP-Version-3.md) | [English](07-VTP-Version-3.en.md) | [日本語](07-VTP-Version-3.ja.md) | [简体中文](07-VTP-Version-3.zh-CN.md) | **Español** | [Français](07-VTP-Version-3.fr.md)

El objetivo de VTP versión 3 sigue siendo sincronizar las VLAN, pero corrige los puntos débiles de las versiones 1 y 2 y añade varias funciones. Las versiones antiguas de IOS para Catalyst no lo admitían; está disponible en las versiones recientes.

> Para los conceptos básicos de VTP (Server / Client / Transparent, revisión), lee primero [06. Introduction to VTP](06-Introduction-to-VTP.es.md).

## Novedades de VTP versión 3

| Función | Descripción |
| --- | --- |
| **Primary Server** | Solo el Primary Server puede crear / modificar / eliminar VLAN → evita el accidente de v1/v2 de **borrar todas las VLAN por error** |
| **VLAN extendidas** | También se sincronizan las VLAN del rango extendido **1006 ~ 4094** |
| **Private VLAN** | Se sincroniza la información de las Private VLAN |
| **VLAN RSPAN** | Se sincronizan las VLAN de Remote SPAN |
| **Soporte de MST** | Se sincroniza la **configuración MST**, que antes había que escribir en cada switch |
| **Autenticación mejorada** | La contraseña puede guardarse como **hidden / secret** |
| **VTP mode off** | VTP se puede **desactivar por completo** de forma global o por interfaz |
| **Compatibilidad** | **Compatible con la versión 2**, no con la versión 1 |

---

## Topología de la práctica

![VTP version 3 lab topology](../images/vtp3-topology.svg)

Todos los enlaces entre SW1, SW2 y SW3 son trunks.

---

## 1. Configuración básica

### 1-1. Primero, el nombre de dominio

```
SW1(config)#vtp version 3
```

VTP versión 3 **no se activa mientras el nombre de dominio esté vacío (NULL).** Configura primero el dominio.

```
SW1(config)#vtp domain JJONG
SW1(config)#vtp version 3
```

Configura SW2 y SW3 de la misma forma. Los tres switches funcionan en el modo por defecto, **VTP Server**.

### 1-2. Ser Server no basta para crear VLAN

```
SW1(config)#vlan 10
```

En VTP versión 3, ni siquiera un Server puede cambiar la base de datos de VLAN si **no es el Primary Server**, así que este comando se rechaza.

### 1-3. Designar el Primary Server

```
SW1#vtp primary vlan
```

- Se ejecuta en **modo privilegiado (`#`), no en modo de configuración**.
- Al confirmar, SW1 pasa a ser el **Primary Server de la función VLAN**, y los demás switches registran que ha cambiado el Primary Server.

```
SW1#show vtp status
```

En `Feature VLAN` se ven `VTP Operating Mode : Primary Server` y el Primary ID (dirección MAC). SW2 y SW3 también muestran **SW1 como Primary Server**.

### 1-4. Ver todos los switches del dominio

```
SW1#show vtp devices
```

Es un comando nuevo de VTP versión 3. A diferencia de CDP, que solo muestra los equipos conectados directamente, muestra **todos los switches del mismo dominio VTP**.

### 1-5. Sincronización de VLAN

```
SW1(config)#vlan 100
```

En SW2 y SW3, `show vlan brief` ya muestra la **VLAN 100**.

### 1-6. VLAN del rango extendido (1006 ~ 4094)

```
SW1(config)#vlan 2000
```

```
SW2#show vlan brief | include 2000
```

Las versiones 1 y 2 no podían enviar VLAN extendidas por VTP, pero en la versión 3 **la VLAN 2000 también se sincroniza**.

---

## 2. Private VLAN

VTP versión 3 sincroniza la información de las Private VLAN. Eso sí, envía **solo la información de la base de datos de VLAN**, no la configuración de los puertos.

```
SW1(config)#vlan 501
SW1(config-vlan)#private-vlan community
SW1(config-vlan)#vlan 502
SW1(config-vlan)#private-vlan isolated
SW1(config-vlan)#vlan 500
SW1(config-vlan)#private-vlan primary
SW1(config-vlan)#private-vlan association 501,502
```

| VLAN | Tipo |
| --- | --- |
| 500 | Primary VLAN |
| 501 | Community VLAN |
| 502 | Isolated VLAN |

```
SW2#show vlan private-vlan
```

La configuración de Private VLAN creada en SW1 aparece **exactamente igual** en SW2 y SW3.

---

## 3. Remote SPAN (RSPAN)

Las VLAN RSPAN también son un tipo especial de VLAN, y ahora VTP las sincroniza.

```
SW1(config)#vlan 600
SW1(config-vlan)#remote-span
```

```
SW3#show vlan remote-span
```

La VLAN 600 aparece como **VLAN RSPAN** en todos los switches.

---

## 4. MST (Multiple Spanning-Tree)

Antes había que escribir a mano la misma configuración MST en cada switch. VTP versión 3 sincroniza MST como una **feature** aparte.

| Feature | Qué sincroniza | Modo por defecto |
| --- | --- | --- |
| **VLAN** | VLAN, VLAN extendidas, Private VLAN, RSPAN | Server |
| **MST** | Configuración MST | Transparent |
| **UNKNOWN** | Reservada para funciones futuras | Transparent |

MST también necesita un **Primary Server**. Puede ser el mismo switch que el Primary de VLAN u otro distinto. Aquí **SW2** será el Primary de MST.

```
SW2(config)#vtp mode server mst
SW2(config)#exit
SW2#vtp primary mst
```

```
SW1(config)#vtp mode client mst
SW3(config)#vtp mode client mst
```

Crea la configuración MST en SW2.

```
SW2(config)#spanning-tree mst configuration
SW2(config-mst)#name JJONG
SW2(config-mst)#revision 1
SW2(config-mst)#instance 1 vlan 10,20
SW2(config-mst)#instance 2 vlan 30,40
```

Activa MST en todos los switches.

```
SW1(config)#spanning-tree mode mst
SW2(config)#spanning-tree mode mst
SW3(config)#spanning-tree mode mst
```

```
SW1#show spanning-tree mst configuration
```

SW1 y SW3 han recibido la configuración MST creada en SW2, y además **queda guardada en la running-config**. Ya no hace falta copiar y pegar la configuración MST.

---

## 5. La feature UNKNOWN

Además de VLAN y MST, `show vtp status` muestra una feature **UNKNOWN**. Está reservada para cuando VTP versión 3 use nuevas funciones en el futuro.

- Por ahora **solo se permite el modo Transparent**.
- `vtp mode server unknown` y `vtp mode client unknown` dan error.

```
SW1(config)#vtp mode transparent unknown
```

---

## 6. Autenticación

En la versión 3, la contraseña puede guardarse como **hidden**.

```
SW1(config)#vtp password <PASSWORD> hidden
```

```
SW1#show vtp password
```

Con hidden, la contraseña solo se muestra como una **cadena hexadecimal cifrada**, no en texto plano, y no se puede extraer la contraseña original del switch.
Al añadir un switch nuevo al dominio, pega esa cadena como **secret**.

```
SW4(config)#vtp password <encrypted-string> secret
```

> Al designar el Primary Server (`vtp primary`) también se pide la contraseña. Una contraseña hidden protege incluso los cambios de Primary Server.

---

## 7. VTP mode off

La versión 3 admite el modo `off`, que **desactiva VTP por completo**.

| Modo | Se sincroniza | Reenvía advertisements |
| --- | --- | --- |
| Transparent | ❌ | ✅ sigue reenviando |
| **Off** | ❌ | ❌ **tampoco reenvía** |

Desactivar de forma global (por feature):

```
SW1(config)#vtp mode off
SW1(config)#vtp mode off vlan
SW1(config)#vtp mode off mst
```

Desactivar por interfaz:

```
SW1(config)#interface GigabitEthernet0/1
SW1(config-if)#no vtp
```

Esta interfaz deja de participar en VTP.

---

## 8. Compatibilidad con versiones anteriores

- VTP versión 3 es **compatible con la versión 2** y **no es compatible con la versión 1.**
- Cuando un switch con versión 3 recibe un advertisement de versión 2, reenvía un advertisement compatible con la versión 2.
- La versión 2 no entiende el nuevo mecanismo de contraseñas, así que al mezclarlas hay que **quitar la contraseña de los switches con versión 3**.
- Si algo no funciona bien, compruébalo con debug. (En la práctica original, un Catalyst 3750 con un IOS antiguo no recibía VTP en absoluto.)

```
SW1#debug sw-vlan vtp events
SW1#no debug all
```

---

## Resumen de comandos

| Comando | Descripción |
| --- | --- |
| `vtp domain <nombre>` | Configura el nombre de dominio (obligatorio en la versión 3) |
| `vtp version 3` | Cambia a VTP versión 3 |
| `vtp primary [vlan \| mst]` | Designa el Primary Server (modo privilegiado) |
| `vtp mode {server \| client \| transparent \| off} [vlan \| mst \| unknown]` | Modo VTP por feature |
| `vtp password <contraseña> [hidden \| secret]` | Configura la contraseña |
| `no vtp` (interface) | Desactiva VTP en esa interfaz |
| `show vtp status` | Muestra el modo, el Primary Server y la revisión |
| `show vtp devices` | Muestra todos los switches VTP del dominio |
| `show vtp password` | Muestra la contraseña (cadena cifrada si es hidden) |

---

## Ideas clave

- VTP versión 3 **exige un nombre de dominio**, y **solo el Primary Server** puede cambiar las VLAN, lo que evita el accidente de "todas las VLAN borradas".
- También sincroniza **VLAN extendidas, Private VLAN, VLAN RSPAN y la configuración MST**.
- Las funciones se dividen en las **features VLAN / MST / UNKNOWN**, cada una con su propio modo y Primary Server.
- Se añadieron las **contraseñas hidden** y el **modo off** (global y por interfaz).
- Es compatible **solo con la versión 2**.
