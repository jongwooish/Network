# How to Troubleshoot VLANs and Trunks

[한국어](../08-How-to-Troubleshoot-VLANs-and-Trunks.md) | [English](08-How-to-Troubleshoot-VLANs-and-Trunks.en.md) | [日本語](08-How-to-Troubleshoot-VLANs-and-Trunks.ja.md) | [简体中文](08-How-to-Troubleshoot-VLANs-and-Trunks.zh-CN.md) | **Español** | [Français](08-How-to-Troubleshoot-VLANs-and-Trunks.fr.md)

Si ya revisaste la configuración de las interfaces (speed / duplex, errores, etc.) y los equipos siguen sin comunicarse, el problema puede estar en las **VLAN o los trunks**. Aquí tienes 5 situaciones habituales con capturas de la práctica.

## Orden de comprobación de un vistazo

| Paso | Qué comprobar | Comando |
| --- | --- | --- |
| 1 | ¿Las interfaces están up / up? | `show ip interface brief`, `show interfaces <if>` |
| 2 | ¿Los dos puertos están en la **misma VLAN**? | `show vlan` |
| 3 | ¿El puerto está en el **modo correcto (access / trunk)**? | `show interfaces <if> switchport` |
| 4 | ¿Port security está bloqueando algo? | `show port-security` |
| 5 | ¿Una **VACL** bloquea el tráfico dentro de la VLAN? | `show vlan filter`, `show vlan access-map` |
| 6 | ¿Los dos extremos del trunk usan la **misma encapsulación**? | `show interfaces <if> switchport` |
| 7 | ¿El trunk **permite** esa VLAN? | `show interfaces <if> trunk` |

---

## 1. Problemas de asignación de VLAN

![Topología con un switch](../images/troubleshooting/topology-single-switch.png)

VPC11 (192.168.1.1/24) no puede hacer ping a VPC12 (192.168.1.2/24). Los equipos están bien; el problema está en el switch.

![El ping falla](../images/troubleshooting/c1-ping-fail.png)

Primero, comprueba si las interfaces tienen problemas.

![show ip int brief](../images/troubleshooting/c1-ip-int-brief.png)

Ambas están up / up. Ahora comprueba la asignación de VLAN.

![show vlan](../images/troubleshooting/c1-show-vlan.png)

`show vlan` muestra **Gi0/0 en la VLAN 1 y Gi0/1 en la VLAN 2**. Están en la misma subred, pero en **VLAN distintas**, así que no se comunican. Pasa Gi0/1 a la VLAN 1.

![Cambio a la VLAN 1](../images/troubleshooting/c1-fix-access-vlan.png)

![El ping funciona](../images/troubleshooting/c1-ping-ok.png)

> **Lección**: Si dos equipos de la misma subred no se comunican, comprueba primero con `show vlan` que **los dos puertos estén en la misma VLAN**.

---

## 2. Problemas de modo switchport

Misma topología, otro problema. Las dos interfaces están up / up. Revisa la asignación de VLAN.

![show vlan](../images/troubleshooting/c2-show-vlan.png)

**Gi0/1 está en la VLAN 10**, pero **Gi0/0 no aparece por ningún lado.** Hay dos posibles causas:
- La interfaz está averiada → no, está up / up
- La interfaz es un **trunk**, no un puerto de acceso (`show vlan` solo muestra los puertos access)

![Información de switchport](../images/troubleshooting/c2-switchport-trunk.png)

Gi0/0 está en **modo trunk** con la VLAN nativa 1. El PC no etiqueta sus tramas con 802.1Q, así que el tráfico de VPC11 acaba en la **VLAN nativa 1** y no llega a VPC12, que está en la VLAN 10. Cámbialo a modo access y ponlo en la VLAN 10.

![Cambio a modo access](../images/troubleshooting/c2-fix-access-mode.png)

![show vlan id 10](../images/troubleshooting/c2-show-vlan-id-10.png)

Ahora los dos puertos están en la VLAN 10. Comprueba también el modo operativo.

![Modo operativo](../images/troubleshooting/c2-operational-mode.png)

![El ping funciona](../images/troubleshooting/c2-ping-ok.png)

> **Lección**: Asegúrate de que el puerto está en el **modo switchport correcto (access / trunk)**. Si un puerto no aparece en `show vlan`, probablemente es un trunk.

---

## 3. Problemas de VACL (VLAN access-list)

Mismos PC, mismo switch y otro problema más. Vamos a comprobar punto por punto.

![show ip int brief](../images/troubleshooting/c3-ip-int-brief.png)

Interfaces up / up. ¿Asignación de VLAN?

![show vlan](../images/troubleshooting/c3-show-vlan.png)

Los dos puertos están en la **VLAN 10**. ¿Y port security?

![port-security](../images/troubleshooting/c3-port-security.png)

Tampoco hay port security. Interfaces bien, misma VLAN, sin port security… si el tráfico sigue bloqueado, sospecha de una **VACL**. Una VACL puede permitir o bloquear el tráfico **dentro de la misma VLAN**.

![show vlan filter](../images/troubleshooting/c3-vlan-filter.png)

La VLAN 10 tiene aplicado un VLAN access-map llamado `BLOCKSTUFF`.

![show vlan access-map](../images/troubleshooting/c3-access-map.png)

![show access-lists](../images/troubleshooting/c3-access-list.png)

- La acción del access-map es **drop**.
- La condición de coincidencia es la access-list 1, y su `permit 192.168.1.0 0.0.0.255` no significa "permitir", sino **"coincide con este rango"**.
- Los dos PC están en ese rango, así que coinciden → **se descartan**.

Cambia la acción a **forward**.

![action forward](../images/troubleshooting/c3-fix-action-forward.png)

![El ping funciona](../images/troubleshooting/c3-ping-ok.png)

> **Lección**: Si todo lo demás parece correcto pero los equipos de la misma VLAN no se comunican, revisa si hay una **VACL**. Un `permit` en la ACL de una VACL significa "coincidencia"; quien decide si se reenvía o se descarta es la **action** del access-map.

---

## 4. Problemas de encapsulación del trunk

Hasta ahora los problemas estaban dentro de un solo switch. Cuando el tráfico tiene que atravesar varios switches, revisa también los **trunks**.

![Topología con dos switches](../images/troubleshooting/topology-two-switches.png)

Cada switch tiene un PC en la VLAN 10, pero no se alcanzan entre sí. Empieza por el puerto de SW1 que va al PC.

![SW1 int g0/0](../images/troubleshooting/c4-sw1-int-g0-0.png)

![SW1 g0/0 switchport](../images/troubleshooting/c4-sw1-g0-0-switchport.png)

![SW1 port-security](../images/troubleshooting/c4-sw1-port-security.png)

Up, access, **VLAN 10**, sin port security. Comprueba lo mismo en SW2.

![SW2 int g0/0](../images/troubleshooting/c4-sw2-int-g0-0.png)

![SW2 g0/0 switchport](../images/troubleshooting/c4-sw2-g0-0-switchport.png)

![SW2 port-security](../images/troubleshooting/c4-sw2-port-security.png)

Los puertos hacia los PC están bien en ambos lados. Ahora hay dos opciones:
- Conectar otro PC de la VLAN 10 al mismo switch y comprobar si se comunican dentro del switch
- Revisar el **enlace entre SW1 y SW2** ← donde más cosas pueden fallar

![SW1 int g0/1](../images/troubleshooting/c4-sw1-int-g0-1.png)

![SW2 int g0/1](../images/troubleshooting/c4-sw2-int-g0-1.png)

Las interfaces están bien. Mira la información de switchport.

![SW1 g0/1 ISL](../images/troubleshooting/c4-sw1-g0-1-isl.png)

![SW2 g0/1 dot1q](../images/troubleshooting/c4-sw2-g0-1-dot1q.png)

Los dos son trunks, pero **SW1 usa ISL y SW2 usa 802.1Q**. Con encapsulaciones distintas, ninguno entiende las tramas del otro. Cambia SW1 a 802.1Q.

![Cambio de encapsulación](../images/troubleshooting/c4-fix-encapsulation.png)

![El ping funciona](../images/troubleshooting/c4-ping-ok.png)

> **Lección**: Los dos extremos de un trunk deben usar la **misma encapsulación**.
> Además, según el modelo de switch, el modo por defecto puede ser `dynamic auto`, y **si ambos extremos están en `dynamic auto`, el enlace queda en access**. Lo más seguro es configurar los trunks de forma explícita con `switchport mode trunk`. ([05. DTP](../05-Dynamic-Trunking-Protocol.md))

---

## 5. Problemas de VLAN permitidas en el trunk

Misma topología. Supón que ya se han comprobado estos puntos y están bien:
- Interfaces (speed / duplex)
- Port security
- Configuración de switchport (asignación de VLAN, modo access)

![El ping falla](../images/troubleshooting/c5-ping-fail.png)

Revisa el trunk entre los switches.

![SW1 g0/1 switchport](../images/troubleshooting/c5-sw1-g0-1-switchport.png)

![SW2 g0/1 switchport](../images/troubleshooting/c5-sw2-g0-1-switchport.png)

Los dos son trunks y los dos usan 802.1Q. Hasta aquí, bien. Pero que el trunk funcione **no significa que todas las VLAN puedan pasar** por él.

![VLAN permitidas en SW1](../images/troubleshooting/c5-sw1-trunk-allowed.png)

![VLAN permitidas en SW2](../images/troubleshooting/c5-sw2-trunk-allowed.png)

`Vlans allowed on trunk` es solo **20**. La VLAN 10 no puede cruzar este trunk. Permite todas las VLAN.

![SW1 allowed vlan all](../images/troubleshooting/c5-sw1-fix-allowed.png)

![SW2 allowed vlan all](../images/troubleshooting/c5-sw2-fix-allowed.png)

![El ping funciona](../images/troubleshooting/c5-ping-ok.png)

> **Lección**: Comprueba siempre si el trunk **permite todas las VLAN** que necesitas.
> En producción, a menudo se permiten solo las VLAN necesarias en lugar de `all`. Para **añadir** una VLAN a la lista existente, usa `switchport trunk allowed vlan add 10`. Si escribes `allowed vlan 10` sin `add`, la lista se **sobrescribe** y puedes cortar otras VLAN.

---

## Resumen de soluciones

| Caso | Causa | Solución |
| --- | --- | --- |
| 1. Asignación de VLAN | Los dos puertos están en VLAN distintas | `switchport access vlan 1` |
| 2. Modo switchport | El puerto del PC es un trunk | `switchport mode access` → `switchport access vlan 10` |
| 3. VACL | La action del access-map es drop | `vlan access-map BLOCKSTUFF 10` → `action forward` |
| 4. Encapsulación del trunk | ISL ↔ 802.1Q no coinciden | `switchport trunk encapsulation dot1q` |
| 5. VLAN permitidas | El trunk solo permite la VLAN 20 | `switchport trunk allowed vlan all` |

## Ideas clave

- Trabaja **de abajo arriba**: interfaz → asignación de VLAN → modo switchport → port security → VACL → trunk.
- Si un puerto no aparece en `show vlan`, puede ser un **trunk**.
- Si los equipos de la misma VLAN siguen bloqueados, sospecha de una **VACL**.
- En los trunks, revisa el **modo y la encapsulación en ambos extremos** y la **lista de VLAN permitidas**.
