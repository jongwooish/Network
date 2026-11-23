# Multiple Spanning-Tree (MST)

[한국어](../01-Multiple-Spanning-Tree.md) | [English](01-Multiple-Spanning-Tree.en.md) | [日本語](01-Multiple-Spanning-Tree.ja.md) | [简体中文](01-Multiple-Spanning-Tree.zh-CN.md) | [Español](01-Multiple-Spanning-Tree.es.md) | **Français**

Par défaut, les commutateurs Cisco Catalyst exécutent **PVST+** ou **Rapid PVST+** (Per VLAN Spanning Tree). Cela signifie que **chaque VLAN a sa propre instance de spanning-tree**. Avec 20 VLAN, 20 spanning trees tournent. Cette page présente **MST (Multiple Spanning Tree)**, qui regroupe plusieurs VLAN dans une seule instance, puis vérifie sa configuration et son fonctionnement en labo.

## Commandes utilisées dans ce labo

| Commande | Usage |
| --- | --- |
| `spanning-tree mode mst` | Activer MST |
| `spanning-tree mst configuration` | Entrer dans le mode de configuration MST |
| `name` / `revision` / `instance N vlan …` | Nom de la région, numéro de révision, correspondance VLAN-instances |
| `spanning-tree mst N priority P` | Définir la bridge priority de l'instance N |
| `show spanning-tree mst configuration` | Vérifier les paramètres de la région et la correspondance |
| `show spanning-tree mst [N]` | Vérifier le root bridge et les rôles de port par instance |

---

## 1. Le problème de PVST+ : un calcul par VLAN

Calculer un spanning tree par VLAN, est-ce un problème ? Comme toujours, **cela dépend.** Prenons un exemple.

![201 VLANs, two root bridges (PVST+)](../images/mst-topology.svg)

Nous avons trois commutateurs et beaucoup de VLAN : de la VLAN 100 à la 300, **201 au total**. Avec PVST ou Rapid PVST, il y a un calcul par VLAN, soit **201 calculs**. Cela consomme beaucoup de CPU et de mémoire.

Si SW2 est le root bridge des VLAN 100–200 et SW3 celui des VLAN 201–300, les topologies sont les suivantes :

![Two outcomes, but 200 spanning-tree instances](../images/mst-pvst-outcomes.svg)

- **VLAN 100–200** : SW2 est le root bridge. Gi0/1 sur SW1 ou Gi0/0 sur SW3 est bloqué. Il y a 100 calculs, mais les résultats sont tous **identiques**.
- **VLAN 201–300** : SW3 est le root bridge. Gi0/0 sur SW1 ou sur SW2 est bloqué.

Il n'y a **que deux résultats**, et pourtant plus de 200 instances de spanning-tree tournent. C'est du gaspillage de cycles CPU et de mémoire.

---

## 2. Concepts de MST : instances et régions

**MST** résout ce problème. Au lieu de calculer un spanning tree par VLAN, on crée des **instances** et on y **associe** les VLAN. Pour le réseau ci-dessus, on pourrait faire ceci :

- Instance 1 : VLAN 100–200
- Instance 2 : VLAN 201–300

Il ne faut plus que **deux calculs (deux instances)** pour toutes ces VLAN.

### Régions MST

MST fonctionne avec la notion de **région**. Les commutateurs qui exécutent MST doivent savoir si leurs voisins exécutent aussi MST.

![MST regions](../images/mst-regions.svg)

Les commutateurs se retrouvent dans la même région quand **tous** ces attributs correspondent. Il peut y avoir une ou plusieurs régions.

| Attribut | Description |
| --- | --- |
| MST configuration name | Un nom qui identifie la région. Vous pouvez l'inventer. |
| MST configuration revision number | Un numéro inventé lui aussi. L'idée est de le changer à chaque modification de la configuration ; il doit seulement être identique sur tous les commutateurs de la région. |
| MST instance to VLAN mapping table | La table qui décide quelle VLAN va dans quelle instance. Vous la configurez vous-même. |

Si les attributs diffèrent, le commutateur est considéré comme situé à la **limite de la région (boundary)**. À la limite, il peut se connecter à une autre région MST ou dialoguer avec un commutateur qui exécute une autre version de spanning tree.

### IST (Internal Spanning Tree)

Dans une région, il y a toujours **une instance de spanning-tree** qui construit une topologie sans boucle. Quand on configure MST, il existe toujours une instance par défaut qui calcule la topologie dans la région : on l'appelle l'**IST (Internal Spanning Tree)**. Par défaut, Cisco utilise l'**instance 0** pour l'IST, et l'IST exécute **rapid spanning tree**.

![MST region: instance 0 (IST)](../images/mst-region-ist.svg)

On peut en plus créer l'instance 1 (VLAN 100–200) et l'instance 2 (VLAN 201–300). Selon le commutateur qui devient root bridge de chaque instance, **un port différent est bloqué**.

![Each instance can block a different port](../images/mst-region-instances.svg)

Un commutateur hors de la région ne voit pas à quoi ressemble la région de l'intérieur. Pour lui, la région MST ressemble à **un seul gros commutateur**, une « boîte noire ».

![Outside the region: one big switch](../images/mst-black-box.svg)

> Le fonctionnement détaillé de MST avec PVST+ est expliqué dans [02. MST and PVST+ Interoperability](../../02-MST-and-PVST-Interoperability/README/02-MST-and-PVST-Interoperability.fr.md).

---

## 3. Configuration de MST

Nous utilisons cette topologie pour le labo.

![Lab topology: one MST region](../images/mst-lab-topology.svg)

Nous créons une région avec ces attributs :

- MST configuration name : **NETWORKLESSONS**
- MST configuration revision number : **1** (un numéro inventé)
- Correspondance VLAN-instances : **Instance 1** = VLAN 10, 20, 30 / **Instance 2** = VLAN 40, 50, 60

D'abord, activons MST sur les trois commutateurs.

![spanning-tree mode mst](../images/mst/01-mode-mst.png)

Regardons l'instance MST par défaut.

![Default MST configuration](../images/mst/02-default-configuration.png)

`show spanning-tree mst configuration` affiche les instances MST. Nous n'en avons encore créé aucune, donc seule l'**instance 0** existe, et toutes les VLAN (1-4094) lui sont associées.

![show spanning-tree mst](../images/mst/03-default-mst.png)

`show spanning-tree mst` affiche aussi la correspondance des VLAN et le root bridge. Avant d'ajouter des instances, faisons le travail préparatoire : configurer en trunk les liens entre les commutateurs.

![Trunks on SW1, SW2, SW3](../images/mst/04-trunks.png)

Créons les VLAN 10 à 60.

![VLANs on SW1, SW2, SW3](../images/mst/05-vlans.png)

Configurons maintenant MST et les instances.

![MST configuration on SW1, SW2, SW3](../images/mst/06-mst-configuration.png)

`spanning-tree mst configuration` permet d'entrer dans le mode de configuration MST ; `name` définit le nom, `revision` le numéro de révision, et la commande `instance` associe les VLAN aux instances. Vérifions.

![Verify the MST configuration](../images/mst/07-verify-configuration.png)

Il y a maintenant trois instances (0, 1, 2). Les VLAN 10, 20, 30 sont dans l'instance 1 et les VLAN 40, 50, 60 dans l'instance 2 ; **toutes les autres VLAN restent dans l'instance 0**.

---

## 4. Changer le root bridge par instance

### Instance 0 (IST) : SW1 comme root bridge

Pour que SW1 soit le root bridge dans la région, modifions la priority de l'**IST (instance 0)**.

![spanning-tree mst 0 priority 4096](../images/mst/08-ist-priority.png)

Vérifions.

![SW1 is the root for the CIST](../images/mst/09-ist-root.png)

SW1 est le root bridge de l'IST. La sortie indique **CIST** (Common and Internal Spanning Tree). Regardons les interfaces.

![Instance 0 interfaces on SW1, SW2, SW3](../images/mst/10-ist-interfaces.png)

Nous connaissons maintenant l'état de toutes les interfaces ; dessinons l'IST.

![Instance 0 (IST): SW1 is the root bridge](../images/mst-ist.svg)

### Instance 1 : SW2 comme root bridge

Ensuite, modifions la priority sur SW2 pour que **SW2** devienne le root bridge de l'instance 1.

![spanning-tree mst 1 priority 4096 on SW2](../images/mst/11-mst1-priority.png)

Cela confirme que SW2 est le root bridge de l'instance 1.

![SW2 is the root for MST1](../images/mst/12-mst1-root.png)

Regardons les interfaces.

![Instance 1 interfaces on SW1, SW2, SW3](../images/mst/13-mst1-interfaces.png)

Voici les rôles de port de l'instance 1. **La topologie est différente de celle de l'instance 0.**

![Instance 1: SW2 is the root bridge](../images/mst-instance1.svg)

### Instance 2 : SW3 comme root bridge

Enfin, faisons de **SW3** le root bridge de l'instance 2.

![spanning-tree mst 2 priority 4096 on SW3](../images/mst/14-mst2-priority.png)

SW3 est maintenant le root bridge de l'instance 2.

![SW3 is the root for MST2](../images/mst/15-mst2-root.png)

Regardons les interfaces.

![Instance 2 interfaces on SW1, SW2, SW3](../images/mst/16-mst2-interfaces.png)

Voici la topologie de l'instance 2.

![Instance 2: SW3 is the root bridge](../images/mst-instance2.svg)

Comparons les instances 1 et 2 côte à côte.

![Instance 1 and instance 2 side by side](../images/mst-instance-compare.svg)

En choisissant un root bridge différent par instance, on obtient des **topologies différentes** :

- **Instance 1** : Gi0/0 sur SW3 est bloqué pour les VLAN 10, 20 et 30.
- **Instance 2** : Gi0/0 sur SW1 est bloqué pour les VLAN 40, 50 et 60.

---

## 5. Et si l'on ajoute un commutateur PVST ?

Que se passe-t-il si l'on ajoute à la topologie SW4, qui exécute PVST ?

![Adding SW4 running PVST](../images/mst-sw4-topology.svg)

Configurons Gi0/2 sur SW2 et SW3 en trunk.

![Trunks on SW2 and SW3](../images/mst/17-sw2-sw3-trunks.png)

Configurons aussi Gi0/0 et Gi0/1 de SW4 en trunk.

![Trunks on SW4](../images/mst/18-sw4-trunks.png)

**PVST** est le mode spanning-tree par défaut des commutateurs Cisco, il n'y a donc rien à changer.

![SW4 runs PVST](../images/mst/19-sw4-pvst.png)

Ajoutons les VLAN 10 à 60 sur SW4.

![VLANs on SW4](../images/mst/20-sw4-vlans.png)

Les trunks vers SW2 et SW3 sont actifs et SW4 connaît toutes les VLAN. Voyons ce que SW4 pense de la topologie, en commençant par la **VLAN 1** (associée à l'instance 0).

![show spanning-tree vlan 1 on SW4](../images/mst/21-sw4-vlan1.png)

SW4 voit **SW1** (503e.ac00.0200) comme root bridge. Gi0/0 est en forwarding (Root) et Gi0/1 en blocking (Alternate).

Voici la **VLAN 10**, associée à l'instance 1.

![show spanning-tree vlan 10 on SW4](../images/mst/22-sw4-vlan10.png)

Nous avons fait de SW2 le root bridge de l'instance 1, mais SW4 voit toujours **SW1** comme root bridge de la VLAN 10. C'est tout à fait normal : MST n'annonce que **les BPDU de l'IST** hors de la région, donc SW4 ne voit rien de l'instance 1 ni de l'instance 2.

> Le texte d'origine (NetworkLessons) indique que SW4 voit SW2 comme root de la VLAN 10, mais l'adresse du Root ID dans la sortie du labo est celle de SW1 (503e.ac00.0200). Cette page suit la sortie du labo.

Vérifions aussi la **VLAN 40**, associée à l'instance 2.

![show spanning-tree vlan 40 on SW4](../images/mst/23-sw4-vlan40.png)

La VLAN 40 appartient à l'instance 2 (root bridge SW3), mais SW4 voit encore **SW1** comme root bridge. SW4 reçoit **la même BPDU pour toutes les VLAN**.

---

## Résumé

- **PVST+ / Rapid PVST+** calculent un spanning tree par VLAN. Avec beaucoup de VLAN, on gaspille CPU et mémoire même quand les résultats sont identiques.
- **MST** associe les VLAN à des **instances** et ne calcule qu'autant d'arbres qu'il y a d'instances.
- Les commutateurs ayant les mêmes **nom, numéro de révision et correspondance VLAN-instances** forment une **région** ; si l'un d'eux diffère, c'est une limite de région.
- Chaque région a toujours l'**IST (instance 0)**, qui exécute rapid spanning tree et apparaît comme **CIST** dans la sortie.
- Avec `spanning-tree mst N priority P`, on peut choisir un root bridge différent par instance, si bien que chaque instance bloque un port différent : c'est la **répartition de charge**.
- Vue de l'extérieur, toute la région ressemble à **un seul commutateur**, et **seules les BPDU de l'IST** sortent de la région. C'est pourquoi un commutateur PVST voit le root bridge de l'IST dans toutes les VLAN.
