# Introduction to Routers and Routing

[한국어](../01-Introduction-to-Routers-and-Routing.md) | [English](01-Introduction-to-Routers-and-Routing.en.md) | [日本語](01-Introduction-to-Routers-and-Routing.ja.md) | [简体中文](01-Introduction-to-Routers-and-Routing.zh-CN.md) | **Español** | [Français](01-Introduction-to-Routers-and-Routing.fr.md)

En esta lección veremos la diferencia entre switches y routers, y explicaré **los fundamentos del enrutamiento**. ¿Qué es un router y qué es exactamente el enrutamiento? Se dice que un switch "conmuta" y un router "enruta", pero ¿qué significa eso en realidad? Empecemos por ahí.

## Switches y routers de un vistazo

| | Switch | Router |
| --- | --- | --- |
| Capa | Capa de enlace de datos (capa 2) | Capa de red (capa 3) |
| Dirección que mira | Dirección MAC de destino | Dirección IP de destino |
| Qué reenvía | Tramas Ethernet | Paquetes IP |
| Tabla que usa | Tabla de direcciones MAC | Tabla de enrutamiento |

---

## 1. ¿En qué se diferencian los switches y los routers?

Un switch **"conmuta" según la información de direcciones MAC**. Solo le importa una cosa: cuando una trama Ethernet entra por una de sus interfaces, mira la **dirección MAC de destino** para decidir adónde enviarla. Es decir, los switches deciden con información de la **capa de enlace de datos (capa 2)**.

Los routers tienen una tarea parecida, pero esta vez miramos **paquetes IP**. Como recordarás, IP está en la **capa de red (capa 3)**. Los routers miran la **dirección IP de destino** de un paquete IP y lo envían por la interfaz correcta.

Quizá estés pensando: ¿cuál es la gran diferencia? Las direcciones MAC y las IP son únicas en cada dispositivo de red, así que ¿por qué no usar MAC en todas partes y conmutar? ¿Por qué hay que mirar direcciones IP y enrutar? Buena pregunta. Te respondo con una imagen.

---

## 2. ¿Por qué necesitamos el enrutamiento?

![Two switches, 400 computers](../images/rt-switching-flat.svg)

Tenemos dos switches, con 200 ordenadores conectados a cada uno. Si los 400 ordenadores quieren comunicarse entre sí, **cada switch tiene que aprender 400 direcciones MAC**: necesita las MAC de los ordenadores de la izquierda y de la derecha.

Ahora piensa en una red realmente grande, por ejemplo **Internet**. ¡Hay millones de dispositivos! ¿Podrías tener millones de entradas en la tabla MAC, una por cada dispositivo de Internet? ¡Imposible! El problema de la conmutación es que **no escala**: no hay jerarquía, solo **direcciones MAC planas de 48 bits**. Veamos el mismo ejemplo con routers.

![Two routers, two networks](../images/rt-routing-networks.svg)

Los 200 ordenadores de la izquierda están conectados a R1 y están en la red **192.168.1.0/24**. R2 tiene 200 ordenadores detrás, y la red que se usa allí es **192.168.2.0/24**. Los routers **"enrutan" según la información IP**. En este ejemplo, R1 solo necesita saber que la red 192.168.2.0/24 está detrás de R2, y R2 solo necesita saber que 192.168.1.0/24 está detrás de R1.

En lugar de una tabla MAC con 400 direcciones, ahora basta con **una sola entrada en cada router para la red del otro**.

---

## 3. La tabla de enrutamiento y el reenvío de paquetes

Igual que los switches usan una **tabla de direcciones MAC** para reenviar tramas Ethernet, los routers usan una **tabla de enrutamiento** para saber adónde reenviar los paquetes IP. Cuando sacas un router nuevo de la caja, crea una tabla de enrutamiento, pero solo contiene las **interfaces directamente conectadas**. Empecemos con un ejemplo sencillo.

![One router, two directly connected networks](../images/rt-one-router.svg)

Arriba tenemos un router y dos ordenadores:

- H1 tiene la dirección IP 192.168.1.1 y usa 192.168.1.254 como puerta de enlace predeterminada.
- H2 tiene la dirección IP 192.168.2.2 y usa 192.168.2.254 como puerta de enlace predeterminada.
- En el router hemos configurado 192.168.1.254 en FastEthernet 0/0 y 192.168.2.254 en FastEthernet 1/0.
- Como también configuramos una máscara de subred con las direcciones IP, el router conoce las direcciones de red y las guarda en su tabla de enrutamiento.

Cuando H1 quiere enviar algo a H2, ocurre esto:

1. H1 envía un paquete IP con la dirección IP de destino 192.168.2.2.
2. H1 comprueba su propia dirección IP y máscara de subred y concluye que 192.168.2.2 está en **otra subred**. Por eso reenvía el paquete IP a su **puerta de enlace predeterminada**.
3. El router recibe el paquete IP, comprueba la dirección IP de destino y revisa su tabla de enrutamiento. 192.168.2.2 **coincide con la entrada 192.168.2.0/24**, así que el router reenvía el paquete IP por su interfaz FastEthernet 1/0.
4. H2 recibe el paquete IP. ¡Todo bien!

---

## 4. Laboratorio: conectar dos redes con un router

Configuremos este escenario en un router real para ver cómo queda. Esta es la topología. En el router del laboratorio (un 2811), la segunda interfaz es Fa0/1.

![Lab topology: PC1 - R1 (2811) - PC2](../images/routing/00-lab-topology.png)

Primero, los ordenadores. Comprueba la dirección IP y la puerta de enlace predeterminada de cada PC.

![ipconfig on PC1 and PC2](../images/routing/01-pc-ipconfig.png)

Ahora configuramos el router: asignamos las direcciones IP a las interfaces y las activamos.

```
R1(config)#int f0/0
R1(config-if)#no shutdown
R1(config-if)#ip address 192.168.1.254 255.255.255.0
R1(config-if)#exit
R1(config)#int f0/1
R1(config-if)#no shut
R1(config-if)#ip add 192.168.2.254 255.255.255.0
R1(config-if)#
```

Eso es todo. Ahora podemos revisar la tabla de enrutamiento.

```
R1#sh ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.1.0/24 is directly connected, FastEthernet0/0
L       192.168.1.254/32 is directly connected, FastEthernet0/0
     192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.2.0/24 is directly connected, FastEthernet0/1
L       192.168.2.254/32 is directly connected, FastEthernet0/1
```

Como ves, el router conoce **las dos redes directamente conectadas** (las entradas `C`). Las entradas `L` son las direcciones de las propias interfaces del router (/32).

Veamos si podemos hacer ping de PC1 a PC2.

![ping from PC1 to PC2](../images/routing/02-ping-pc1-to-pc2.png)

¡Excelente, el ping funciona! Acabamos de enrutar nuestro primer paquete IP. El TTL es 127 porque bajó en uno al pasar por el router.

> Cuando hagas ping de un ordenador Windows a otro, ten en cuenta que **el firewall puede bloquear el tráfico ICMP**.

---

## 5. Ver la ruta con traceroute

Cuando accedes a un servidor de Internet, pasas por **muchos routers** hasta llegar al destino. Puedes ver esa ruta con **traceroute** (`tracert` en Windows). Este es el resultado de trazar la ruta desde un PC hasta `www.cisco.com`:

![tracert www.cisco.com](../images/routing/03-tracert.png)

> El original explica que en Internet real la traza hasta www.cisco.com pasa por cuatro routers y muestra sus nombres. En la captura del laboratorio, www.cisco.com se resuelve como 192.168.3.1 en la red del laboratorio: la traza pasa por 192.168.0.254 y 192.168.2.1 (el salto 2 no responde), llega al destino en el salto 4 y no muestra nombres de routers.

Cada salto muestra la dirección IP de un router del camino. Si el ordenador hace una búsqueda del nombre de host (hostname lookup), también se ven los nombres de los routers. Traceroute en Windows usa **ICMP**.

En otra lección veremos el enrutamiento estático y el dinámico.

---

## Resumen

- Un **switch** reenvía tramas según la **dirección MAC** de destino en la capa 2; un **router** reenvía paquetes según la **dirección IP** de destino en la capa 3.
- Las direcciones MAC son planas, de 48 bits y sin jerarquía, así que **no escalan**. Un router usa una sola entrada por red (por ejemplo 192.168.2.0/24) para representar muchos dispositivos.
- La tabla de enrutamiento de un router nuevo solo tiene **redes directamente conectadas**. Al configurar una dirección IP y una máscara en una interfaz se crean las entradas `C` (connected) y `L` (local).
- Si el destino está en **otra subred**, el host envía el paquete a su **puerta de enlace predeterminada**; el router busca la entrada que coincide en su tabla y lo envía por esa interfaz.
- Verifica con `show ip route` y, en el PC, con `ipconfig`, `ping` y `tracert`. El firewall de Windows puede bloquear ICMP.
