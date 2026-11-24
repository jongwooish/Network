# Spanning-Tree BPDU Guard

[한국어](../03-BPDU-Guard.md) | [English](03-BPDU-Guard.en.md) | [日本語](03-BPDU-Guard.ja.md) | [简体中文](03-BPDU-Guard.zh-CN.md) | **Español** | [Français](03-BPDU-Guard.fr.md)

**BPDU Guard** es una de las funciones que ayudan a proteger la topología de spanning-tree. Cuando **llega una BPDU a una interfaz con BPDU Guard, la interfaz pasa a err-disable**. Veamos con un ejemplo por qué lo querrías.

## Cómo configurar BPDU Guard

| Dónde | Comando | Se aplica a |
| --- | --- | --- |
| Interfaz | `spanning-tree bpduguard enable` | Esa interfaz |
| Global | `spanning-tree portfast bpduguard default` | Todas las interfaces con PortFast activado |

---

## 1. ¿Por qué necesitamos BPDU Guard?

![An attacker sends superior BPDUs](../images/bpduguard-attack.svg)

En la topología anterior, spanning tree funciona perfectamente. Por defecto, spanning tree **envía y recibe BPDU en todas las interfaces**. En este ejemplo hay un ordenador conectado a fa0/2 de SW2. Alguien con intenciones curiosas u hostiles podría ejecutar una herramienta que genere **BPDU con un bridge ID superior**. Los switches creerían entonces que el root bridge se alcanza ahora a través de SW2, y spanning tree se **recalcularía**. No suena bien, ¿verdad? Esto es lo que podría salir mal:

![Man-in-the-middle: the attacker becomes the root](../images/bpduguard-mitm.svg)

Incluso se podría hacer un **ataque man-in-the-middle** sin que nadie lo supiera. Imagina que el atacante conecta un ordenador a dos switches y se convierte en root bridge: todo el tráfico entre los switches pasa ahora por el ordenador del atacante. Basta con abrir Wireshark y esperar a que ocurra la magia.

BPDU Guard lo impide, porque bloquea la interfaz en cuanto llega una BPDU.

![BPDU Guard drops the attacker](../images/bpduguard-block.svg)

Cuando BPDU Guard recibe una BPDU en una interfaz, pone la interfaz en modo **err-disable**. Veamos cómo configurarlo.

---

## 2. Configurarlo en una interfaz

Usaremos esta topología.

![Lab topology: SW2, SW3](../images/bpdu-guard/01-topology.png)

Dos switches bastan para demostrar BPDU Guard. Configuraré Gi0/0 de SW2 para que pase a err-disable si recibe una BPDU de SW3.

![spanning-tree bpduguard enable on Gi0/0](../images/bpdu-guard/02-bpduguard.png)

Así se activa en la interfaz. En la vida real nunca harías esto entre switches; configúralo en las **interfaces en modo access conectadas a ordenadores**.

![BLOCK_BPDUGUARD and err-disable](../images/bpdu-guard/03-err-disable.png)

Como era de esperar, ahí va nuestra interfaz. Cuando Gi0/0 recibe una BPDU, aparece el mensaje `BLOCK_BPDUGUARD`, el puerto pasa a **err-disable** y el line protocol cae. Quitamos BPDU Guard y recuperamos la interfaz con `shutdown` / `no shutdown`.

![no spanning-tree bpduguard, shutdown, no shutdown](../images/bpdu-guard/04-remove.png)

---

## 3. Configurarlo de forma global

En lugar de activar BPDU Guard por interfaz, también puedes activarlo **de forma global**. El equipo del laboratorio (IOSv) no tiene este comando, así que esta parte se hizo en **Packet Tracer**.

```
SW2(config)#spanning-tree portfast bpduguard default
```

Esto activa BPDU Guard en **todas las interfaces con PortFast activado**, por eso normalmente PortFast también se activa de forma global:

```
SW2(config)#spanning-tree portfast default
```

Este es un comando útil para comprobar la configuración:

```
SW2#sh spanning-tree summary
Switch is in pvst mode
Root bridge for:
Extended system ID           is enabled
Portfast Default             is enabled
PortFast BPDU Guard Default  is enabled
Portfast BPDU Filter Default is disabled
Loopguard Default            is disabled
EtherChannel misconfig guard is disabled
UplinkFast                   is disabled
BackboneFast                 is disabled
Configured Pathcost method used is short
```

Se ve que **Portfast Default** y **PortFast BPDU Guard Default** están activados de forma global.

---

## Resumen

- Por defecto todas las interfaces envían y reciben BPDU, así que una **BPDU superior** enviada desde un puerto de host provoca un recálculo y puede convertir al atacante en root bridge, lo que permite un **ataque man-in-the-middle**.
- **BPDU Guard** pone en **err-disable** la interfaz que recibe una BPDU (mensajes `BLOCK_BPDUGUARD`, `ERR_DISABLE`).
- Interfaz: `spanning-tree bpduguard enable` / global: `spanning-tree portfast bpduguard default` (todos los puertos PortFast).
- Úsalo en **puertos access conectados a ordenadores**, no en enlaces entre switches. Un puerto en err-disable se recupera con `shutdown` / `no shutdown` tras eliminar la causa.
- `show spanning-tree summary` muestra la configuración global de PortFast y BPDU Guard.
