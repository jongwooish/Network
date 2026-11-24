# Spanning-Tree PortFast

[한국어](../01-PortFast.md) | [English](01-PortFast.en.md) | [日本語](01-PortFast.ja.md) | [简体中文](01-PortFast.zh-CN.md) | **Español** | [Français](01-PortFast.fr.md)

**PortFast** es una función propietaria de Cisco para lidiar con los cambios de topología de spanning-tree. Si no sabes cómo reacciona spanning tree a los cambios de topología, lee primero [Spanning-Tree Topology Change Notification](../../../02-RSTP/03-STP/README/05-Spanning-Tree-Topology-Change-Notification.es.md). Te ayudará a entender por qué necesitamos PortFast.

## Qué hace PortFast

| Comportamiento | Descripción |
| --- | --- |
| Directo a forwarding | Una interfaz con PortFast se salta los estados listening y learning y pasa a **forwarding de inmediato**. |
| Sin TCN | Cuando una interfaz con PortFast sube o baja, el switch **nunca genera una TCN (topology change notification)**. |

Las interfaces conectadas a hosts (PC, servidores…) suben y bajan constantemente, así que conviene activar PortFast en ellas. En cambio, **no lo actives en interfaces conectadas a otro hub o switch.**

---

## 1. Topología del laboratorio

Compararemos una interfaz con y sin PortFast usando esta topología.

![Lab topology: VPC, SW1, SW2](../images/portfast/01-topology.png)

Hay dos switches y un host (VPC) conectado a Gi0/0 de SW1. El único motivo del segundo switch es que SW1 tenga otro switch al que enviar las notificaciones de cambio de topología. Empecemos sin PortFast.

---

## 2. PortFast desactivado

Para ver lo interesante, activamos un debug en SW1.

![debug spanning-tree events](../images/portfast/02-debug.png)

Al enchufar el cable que conecta el host a SW1, ocurre esto:

![Gi0/0: listening → learning → forwarding](../images/portfast/03-without-portfast.png)

Es el comportamiento normal de spanning tree. Gi0/0 pasa por **listening → learning → forwarding**, así que tarda dos forward delay de 15 segundos, unos 30 segundos, en reenviar.

Cada vez que desenchufo el cable, spanning tree genera una TCN. Hay un comando útil para ver cuántas se han enviado hasta ahora.

![show spanning-tree detail: 5 topology changes](../images/portfast/04-topology-changes.png)

Ha habido **cinco** cambios de topología en la VLAN 1. Desenchufemos el cable del host.

> El texto original dice "cuatro", pero la salida del laboratorio muestra `Number of topology changes 5`.

![Unplug: Generating TC trap for port Gi0/0](../images/portfast/05-unplug.png)

Spanning tree envía una TCN por la interfaz hacia SW2 (Generating TC trap) y el contador aumenta.

![Number of topology changes 6](../images/portfast/06-topology-changes.png)

En resumen, cada vez que desenchufamos el cable, el switch genera una **TCN**. Veamos qué cambia al activar PortFast.

---

## 3. PortFast activado

Basta con activar PortFast en la interfaz GigabitEthernet0/0 que conecta el host.

![spanning-tree portfast on Gi0/0](../images/portfast/07-portfast.png)

Aparece un gran **aviso** de que PortFast no debe usarse en interfaces conectadas a otros switches, etc. También aparece un mensaje de que PortFast solo tiene efecto en **modo non-trunking (access)**.

En lugar de activarlo interfaz por interfaz, también hay un comando global. El resultado es el mismo, pero activa PortFast en **todas las interfaces en modo access** de una vez.

```
SW1(config)#spanning-tree portfast default
```

Conectemos el host de nuevo.

![Gi0/0 jumps to forwarding from blocking](../images/portfast/08-with-portfast.png)

La interfaz se salta listening y learning y pasa **directamente a forwarding** (`jump to forwarding from blocking`). Y ahora, al desenchufar este cable, el switch ya no genera ninguna TCN.

---

## Resumen

- **PortFast** pone una interfaz **directamente en forwarding** sin listening ni learning, y **no se genera TCN** cuando esa interfaz sube o baja.
- Sin PortFast, incluso un puerto de host pasa unos 30 segundos en listening y learning, y cada desconexión envía una TCN y aumenta `Number of topology changes`.
- Se activa con `spanning-tree portfast` en una interfaz, o con `spanning-tree portfast default` de forma global (todos los puertos access).
- Actívalo **solo en puertos de host**, nunca en puertos conectados a hubs o switches; si lo haces, pueden aparecer bucles temporales.
