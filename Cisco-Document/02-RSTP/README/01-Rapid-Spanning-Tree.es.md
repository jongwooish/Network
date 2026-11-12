# Rapid Spanning-Tree (RSTP)

[한국어](../01-Rapid-Spanning-Tree.md) | [English](01-Rapid-Spanning-Tree.en.md) | [日本語](01-Rapid-Spanning-Tree.ja.md) | [简体中文](01-Rapid-Spanning-Tree.zh-CN.md) | **Español** | [Français](01-Rapid-Spanning-Tree.fr.md)

Cada vez hay más enrutamiento en nuestras redes. Protocolos de enrutamiento como OSPF y EIGRP reaccionan a los cambios de la red mucho más rápido que spanning tree. Para seguirles el ritmo se creó **Rapid Spanning Tree (RSTP, IEEE 802.1w)**.

RSTP es **una evolución, no una revolución** del spanning tree original. Se cambiaron algunas cosas internas para que sea más rápido, pero **la configuración es la misma**. A partir de aquí, al spanning tree original (802.1D) lo llamaremos **classic spanning tree**.

## Diferencias de un vistazo

| Aspecto | Classic STP (802.1D) | RSTP (802.1w) |
| --- | --- | --- |
| Estados de puerto | Blocking / Listening / Learning / Forwarding | **Discarding** / Learning / Forwarding |
| Roles de puerto | Root, Designated, Alternate (bloqueado) | Root, Designated, Alternate, **Backup** |
| BPDU | Solo 2 bits de flags (TC, TCA) | **BPDU versión 2**, todos los flags en uso |
| Quién envía BPDU | Solo el root; los demás reenvían | **Todos los switches, cada hello time (2 s)** |
| Detección de fallos | Max age 20 s | **3 BPDU perdidas** (keepalive) |
| BPDU inferiores | Se ignoran | **Se aceptan** (como BackboneFast) |
| Paso a forwarding | Por temporizadores (listening → learning, ~30 s) | **Negociación proposal / agreement (sync)** |
| PortFast | PortFast | **Edge port** |
| UplinkFast | Hay que configurarlo | **Incluido por defecto** |
| Cambio de topología | TCN hasta el root, el root avisa | **El switch que lo detecta lo inunda enseguida** |

---

## 1. Estados de puerto: 4 → 3

![Port states](../images/rstp-port-states.svg)

RSTP solo tiene tres estados de puerto: **discarding, learning y forwarding**. **Discarding** une los estados blocking y listening del classic.

| Classic Spanning Tree | Rapid Spanning Tree | ¿Puerto activo en la topología? | ¿Aprende MAC? |
| --- | --- | --- | --- |
| Blocking | Discarding | No | No |
| Listening | Discarding | Yes | No |
| Learning | Learning | Yes | Yes |
| Forwarding | Forwarding | Yes | Yes |

---

## 2. Roles de puerto

![Port roles](../images/rstp-port-roles.svg)

- **Root bridge**: el switch con el mejor bridge ID (priority + MAC). Aquí es SW1 (MAC AAA).
- **Root port**: el puerto de un switch non-root con el menor coste hasta el root.
- **Designated port**: solo uno por segmento. Lo es el puerto que **puede enviar la mejor BPDU**.
- **Alternate port**: Fa0/16 de SW3 recibe mejores BPDU de SW2, así que se bloquea. Es un camino de respaldo para el root port.

Hasta aquí todo es igual que en classic spanning tree. El rol nuevo en RSTP es el **backup port**.

### Backup port

![Backup port](../images/rstp-backup-port.svg)

Hemos puesto un **hub** entre SW2 y SW3. Sin el hub, Fa0/16 y Fa0/17 de SW2 serían los dos designated ports.

Por el hub, Fa0/16 y Fa0/17 de SW2 están ahora en **el mismo dominio de colisión**. SW2 recibe sus propias BPDU en Fa0/16 y Fa0/17, así que sabe que tiene dos conexiones al mismo segmento.

- Fa0/16 → **designated port**
- Fa0/17 → **backup port** de Fa0/16

> Es muy poco probable ver un backup port en una red de producción. Si quitas el hub, los dos vuelven a ser designated ports.

---

## 3. BPDU (versión 2)

![RSTP BPDU](../images/rstp-bpdu-flags.svg)

Classic spanning tree solo usaba **dos bits** del campo flags:
- Topology Change
- Topology Change Acknowledgement

RSTP usa **todos los bits del campo flags**. En especial, el campo **port role** indica el rol del puerto que envió la BPDU:
- Unknown
- Alternate / backup port
- Root port
- Designated port

Esta nueva BPDU se llama **BPDU versión 2**. Los switches que solo ejecutan classic spanning tree la descartan.

---

## 4. Otros cambios

### Todos los switches envían BPDU cada hello time
En classic spanning tree **solo el root bridge** generaba BPDU y los switches non-root solo reenviaban las que recibían por su root port. En RSTP **todos los switches generan BPDU cada 2 segundos (hello time)**. El hello time se puede cambiar.

### BPDU = keepalive
Classic spanning tree esperaba el **max age (20 s)** antes de descartar una BPDU. RSTP usa las BPDU como **keepalive**, igual que los hello de OSPF o EIGRP. Si un switch **pierde tres BPDU seguidas** de un vecino, da por perdida la conexión y borra todas las direcciones MAC de inmediato.

### Acepta BPDU inferiores
Classic spanning tree ignora las BPDU inferiores; RSTP las acepta. Es prácticamente la función **BackboneFast** de classic spanning tree.

### Negociación en vez de temporizadores
Classic spanning tree tenía que pasar por listening y learning antes de forwarding, lo que tardaba **30 segundos** con los temporizadores por defecto. RSTP decide si un puerto puede pasar a forwarding mediante **negociación**, no con temporizadores. Este es el cambio más importante: **convergencia rápida**.

### PortFast = edge port
Con PortFast en classic spanning tree, el puerto se salta listening y learning y pasa directamente a forwarding, y no genera cambios de topología al subir o bajar. RSTP sigue usando PortFast, pero lo llama **edge port**.

### Tipo de enlace
RSTP solo puede pasar puertos a forwarding rápidamente en **edge ports** y enlaces **point-to-point**.

| Tipo de enlace | Dúplex | Descripción |
| --- | --- | --- |
| Point-to-point | Full duplex | Enlaces normales entre switches |
| Shared | Half duplex | Enlaces con un hub en medio |

---

## 5. Sync: proposal / agreement

![RSTP sync overview](../images/rstp-sync-overview.svg)

SW1 es el root bridge; SW2, SW3 y SW4 son non-root.

1. Cuando sube el enlace SW1–SW2, los dos puertos empiezan **bloqueados**. Cuando SW2 recibe la BPDU de SW1, comienza una negociación llamada **sync**.
2. En cuanto SW2 recibe la BPDU del root, **bloquea todos sus designated ports non-edge** (non-edge = puertos hacia otros switches; edge = puertos con PortFast). Entonces el enlace SW1–SW2 pasa a **forwarding**.
3. Después SW2 hace sync con SW3 y SW4 para que esos enlaces también pasen rápido a forwarding.

Estos son exactamente los mensajes que se intercambian, paso a paso:

![Proposal and agreement](../images/rstp-proposal-agreement.svg)

| Paso | Qué ocurre |
| --- | --- |
| 1. Proposal | SW1 activa el **bit proposal** en los flags de la BPDU. SW2 sabe que SW1 es el root |
| 2. Sync | SW2 **bloquea todos los puertos non-edge** y se prepara para sincronizar con SW3 y SW4 |
| 3. Agreement | SW2 envía un **agreement** (copia de la BPDU proposal con proposal OFF y agreement ON). Fa0/14 de SW2 pasa a forwarding |
| 4. Forwarding | Al recibir el agreement, SW1 pone Fa0/14 en **forwarding de inmediato** |
| 5. Proposal | SW2 envía el mismo proposal por Fa0/16 (SW3) y Fa0/19 (SW4) |
| 6. Agreement | SW3 y SW4 no tienen más puertos, así que responden con agreement enseguida |
| 7. Forwarding | SW2 pone Fa0/16 y Fa0/19 en forwarding — listo |

> Son solo unos pocos mensajes de ida y vuelta, así que es **mucho más rápido** que el mecanismo por temporizadores del classic (listening → learning → forwarding).

---

## 6. UplinkFast viene incluido

En classic spanning tree había que activar **UplinkFast** a mano. RSTP lo trae por defecto. Cuando un switch pierde su root port, **pone su alternate port en forwarding de inmediato**.

El UplinkFast clásico necesitaba tramas multicast para actualizar las tablas MAC de todos los switches. RSTP no lo necesita porque su mecanismo de cambio de topología es distinto.

---

## 7. Cambio de topología

![Topology change](../images/rstp-topology-change.svg)

En classic spanning tree, **un fallo de enlace provocaba un cambio de topología**. En RSTP, un fallo de enlace no se considera cambio de topología. **Solo cuenta cuando un puerto non-edge (hacia otro switch) pasa a forwarding**.

El switch que **detecta** el cambio de topología:
- Inicia un **topology change while timer** en todos sus designated ports non-edge y en su root port. El valor es **el doble del hello time**.
- **Borra las direcciones MAC** aprendidas en esos puertos.
- Mientras el temporizador está activo, activa el **bit TC** en las BPDU que envía por esos puertos. También envía BPDU por su root port.

Un **switch vecino que recibe** una BPDU con el bit TC activado:
- Borra las direcciones MAC de **todos los puertos excepto** el puerto por donde la recibió.
- Inicia su propio topology change while timer y envía BPDU con el bit TC activado por todos sus designated ports y su root port.

> En lugar de subir hasta el root bridge como en classic spanning tree, el cambio de topología se **inunda rápidamente por toda la red**.

---

## 8. Compatibilidad con classic spanning tree

![Compatibility](../images/rstp-compatibility.svg)

RSTP y classic spanning tree **son compatibles**. Sin embargo, donde un switch RSTP habla con uno classic, **dejan de funcionar todas las funciones que hacen rápido a RSTP**.

En la imagen, entre SW1 y SW2 se usa RSTP, pero entre SW2 y SW3 se **vuelve a classic spanning tree**.

---

## Ideas clave

- RSTP (802.1w) es una **evolución** de classic STP. Misma configuración, funcionamiento interno más rápido.
- Tres estados: **discarding / learning / forwarding**. Rol nuevo: **backup port**.
- **BPDU versión 2**: usa todos los flags e incluye el port role. Todos los switches la envían cada 2 s; 3 perdidas = fallo.
- Converge con **proposal → sync → agreement** en vez de temporizadores. Las transiciones rápidas solo se dan en **edge ports** y enlaces **point-to-point**.
- **UplinkFast incluido**. El cambio de topología ocurre **solo cuando un puerto non-edge pasa a forwarding** y se propaga rápido por **inundación**.
- Compatible con classic STP, pero el tramo compartido funciona como classic.
