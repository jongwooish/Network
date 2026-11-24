# Spanning-Tree Root Guard

[한국어](../02-Root-Guard.md) | [English](02-Root-Guard.en.md) | [日本語](02-Root-Guard.ja.md) | [简体中文](02-Root-Guard.zh-CN.md) | **Español** | [Français](02-Root-Guard.fr.md)

**Root Guard** se asegura de que no aceptes a cierto switch como root bridge. Las BPDU se envían y procesan con normalidad, pero si un switch envía de repente una BPDU con un **bridge ID superior**, no lo aceptarás como root bridge. Normalmente el switch con el mejor bridge ID se convertiría en root bridge, pero eso no ocurre en un puerto con Root Guard.

## Root Guard en pocas palabras

| Situación | Comportamiento |
| --- | --- |
| Normalmente | Las BPDU se envían y reciben como siempre. |
| BPDU superior recibida en un puerto con Root Guard | El puerto se bloquea como **root-inconsistent** para esa VLAN. |
| Dejan de llegar BPDU superiores | El puerto se recupera automáticamente. |

---

## 1. Topología del laboratorio

Usaremos esta topología.

![Lab topology: VPC, SW2, SW3](../images/root-guard/01-topology.png)

La configuración usa SW2 y SW3. Primero nos aseguramos de que SW3 **no** sea el root bridge bajando la priority de SW2 para que SW2 sea el root bridge.

![spanning-tree vlan 1 priority 4096 on SW2](../images/root-guard/02-sw2-priority.png)

---

## 2. Configurar Root Guard

Activamos Root Guard en la interfaz de SW2 hacia SW3.

![spanning-tree guard root on Gi0/0](../images/root-guard/03-root-guard.png)

Aparece un mensaje (`ROOTGUARD_CONFIG_CHANGE`) de que Root Guard se ha activado.

> El original usa Fastethernet0/16 y el dibujo de la topología marca el enlace SW2–SW3 como Gi0/1, pero en la salida del laboratorio el puerto hacia SW3 es **Gi0/0**. Esta página sigue la salida del laboratorio.

Activamos un debug para ver qué pasa.

![debug spanning-tree events on SW2](../images/root-guard/04-debug.png)

---

## 3. ¿Qué pasa con una BPDU superior?

Ahora vamos a "molestar" a SW2 cambiando la priority de SW3 al valor más bajo posible (0). Normalmente SW3 debería convertirse en root bridge.

![spanning-tree vlan 1 priority 0 on SW3](../images/root-guard/05-sw3-priority.png)

¿Qué opina SW2 de esto?

![ROOTGUARD_BLOCK on SW2](../images/root-guard/06-root-guard-block.png)

SW2 oye un root mejor (`1-505b.2e00.0200`) en Gi0/0, pero **no acepta** a SW3 como root bridge. En su lugar, bloquea la interfaz para esta VLAN (`ROOTGUARD_BLOCK`). Este es otro comando útil para comprobarlo:

![show spanning-tree inconsistentports](../images/root-guard/07-inconsistentports.png)

Nos dice que GigabitEthernet0/0 está **Root Inconsistent** en la VLAN 1.

Root Guard es útil en los **switches de núcleo o de distribución**, para que los switches que están por debajo nunca sean elegidos root bridge.

---

## Resumen

- **Root Guard** se niega a aceptar un nuevo root bridge a partir de una BPDU superior recibida en el puerto, y bloquea el puerto como **root-inconsistent**.
- Se configura con `spanning-tree guard root` en la interfaz. Al activarlo aparece `ROOTGUARD_CONFIG_CHANGE`, y al bloquear, `ROOTGUARD_BLOCK`.
- `show spanning-tree inconsistentports` muestra el puerto bloqueado y la VLAN (`Root Inconsistent`).
- Actívalo en los switches de núcleo/distribución, en los **puertos hacia los switches inferiores (access)**, para proteger la ubicación del root bridge.
