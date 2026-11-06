# VTP Version 3

[한국어](../07-VTP-Version-3.md) | [English](07-VTP-Version-3.en.md) | [日本語](07-VTP-Version-3.ja.md) | [简体中文](07-VTP-Version-3.zh-CN.md) | [Español](07-VTP-Version-3.es.md) | **Français**

L'objectif de VTP version 3 reste la synchronisation des VLAN, mais il corrige les faiblesses des versions 1 et 2 et ajoute plusieurs fonctions. Les anciens IOS Catalyst ne le prenaient pas en charge ; il est disponible dans les versions récentes.

> Pour les bases de VTP (Server / Client / Transparent, révision), lisez d'abord [06. Introduction to VTP](06-Introduction-to-VTP.fr.md).

## Les nouveautés de VTP version 3

| Fonction | Description |
| --- | --- |
| **Primary Server** | Seul le Primary Server peut créer / modifier / supprimer des VLAN → évite l'accident de v1/v2 où **tous les VLAN sont effacés par erreur** |
| **VLAN étendus** | Les VLAN de la plage étendue **1006 ~ 4094** sont aussi synchronisés |
| **Private VLAN** | Les informations de Private VLAN sont synchronisées |
| **VLAN RSPAN** | Les VLAN Remote SPAN sont synchronisés |
| **Prise en charge de MST** | La **configuration MST**, autrefois saisie sur chaque switch, est synchronisée |
| **Authentification renforcée** | Le mot de passe peut être stocké en **hidden / secret** |
| **VTP mode off** | VTP peut être **complètement désactivé** globalement ou par interface |
| **Compatibilité** | **Compatible avec la version 2**, pas avec la version 1 |

---

## Topologie du TP

![VTP version 3 lab topology](../images/vtp3-topology.svg)

Tous les liens entre SW1, SW2 et SW3 sont des trunks.

---

## 1. Configuration de base

### 1-1. D'abord le nom de domaine

```
SW1(config)#vtp version 3
```

VTP version 3 **ne s'active pas tant que le nom de domaine est vide (NULL).** Définissez d'abord le domaine.

```
SW1(config)#vtp domain JJONG
SW1(config)#vtp version 3
```

Configurez SW2 et SW3 de la même manière. Les trois switchs fonctionnent dans le mode par défaut, **VTP Server**.

### 1-2. Être Server ne suffit pas pour créer des VLAN

```
SW1(config)#vlan 10
```

Avec VTP version 3, même un Server ne peut pas modifier la base de données VLAN s'il **n'est pas le Primary Server** : cette commande est donc refusée.

### 1-3. Désigner le Primary Server

```
SW1#vtp primary vlan
```

- À exécuter en **mode privilégié (`#`), pas en mode configuration**.
- Après confirmation, SW1 devient le **Primary Server de la fonction VLAN**, et les autres switchs journalisent le changement de Primary Server.

```
SW1#show vtp status
```

Sous `Feature VLAN`, on voit `VTP Operating Mode : Primary Server` et le Primary ID (adresse MAC). SW2 et SW3 indiquent eux aussi **SW1 comme Primary Server**.

### 1-4. Voir tous les switchs du domaine

```
SW1#show vtp devices
```

Une nouvelle commande de VTP version 3. Contrairement à CDP, qui n'affiche que les équipements directement connectés, elle affiche **tous les switchs du même domaine VTP**.

### 1-5. Synchronisation des VLAN

```
SW1(config)#vlan 100
```

Sur SW2 et SW3, `show vlan brief` affiche maintenant le **VLAN 100**.

### 1-6. VLAN de la plage étendue (1006 ~ 4094)

```
SW1(config)#vlan 2000
```

```
SW2#show vlan brief | include 2000
```

Les versions 1 et 2 ne pouvaient pas transmettre les VLAN étendus par VTP, mais avec la version 3 **le VLAN 2000 est aussi synchronisé**.

---

## 2. Private VLAN

VTP version 3 synchronise les informations de Private VLAN. Il n'envoie toutefois **que les informations de la base de données VLAN**, pas la configuration des ports.

```
SW1(config)#vlan 501
SW1(config-vlan)#private-vlan community
SW1(config-vlan)#vlan 502
SW1(config-vlan)#private-vlan isolated
SW1(config-vlan)#vlan 500
SW1(config-vlan)#private-vlan primary
SW1(config-vlan)#private-vlan association 501,502
```

| VLAN | Type |
| --- | --- |
| 500 | Primary VLAN |
| 501 | Community VLAN |
| 502 | Isolated VLAN |

```
SW2#show vlan private-vlan
```

La configuration Private VLAN créée sur SW1 apparaît **exactement à l'identique** sur SW2 et SW3.

---

## 3. Remote SPAN (RSPAN)

Les VLAN RSPAN sont aussi un type particulier de VLAN, et VTP les synchronise désormais.

```
SW1(config)#vlan 600
SW1(config-vlan)#remote-span
```

```
SW3#show vlan remote-span
```

Le VLAN 600 apparaît comme **VLAN RSPAN** sur tous les switchs.

---

## 4. MST (Multiple Spanning-Tree)

Autrefois, il fallait saisir à la main la même configuration MST sur chaque switch. VTP version 3 synchronise MST comme une **feature** séparée.

| Feature | Ce qui est synchronisé | Mode par défaut |
| --- | --- | --- |
| **VLAN** | VLAN, VLAN étendus, Private VLAN, RSPAN | Server |
| **MST** | Configuration MST | Transparent |
| **UNKNOWN** | Réservée aux fonctions futures | Transparent |

MST a aussi besoin d'un **Primary Server**. Il peut s'agir du même switch que le Primary VLAN ou d'un autre. Ici, **SW2** devient le Primary MST.

```
SW2(config)#vtp mode server mst
SW2(config)#exit
SW2#vtp primary mst
```

```
SW1(config)#vtp mode client mst
SW3(config)#vtp mode client mst
```

Créez la configuration MST sur SW2.

```
SW2(config)#spanning-tree mst configuration
SW2(config-mst)#name JJONG
SW2(config-mst)#revision 1
SW2(config-mst)#instance 1 vlan 10,20
SW2(config-mst)#instance 2 vlan 30,40
```

Activez MST sur tous les switchs.

```
SW1(config)#spanning-tree mode mst
SW2(config)#spanning-tree mode mst
SW3(config)#spanning-tree mode mst
```

```
SW1#show spanning-tree mst configuration
```

SW1 et SW3 ont reçu la configuration MST créée sur SW2, et elle est aussi **enregistrée dans la running-config**. Plus besoin de copier-coller la configuration MST.

---

## 5. La feature UNKNOWN

En plus de VLAN et MST, `show vtp status` affiche une feature **UNKNOWN**. Elle est réservée au jour où VTP version 3 utilisera de nouvelles fonctions.

- Pour l'instant, **seul le mode Transparent** est autorisé.
- `vtp mode server unknown` et `vtp mode client unknown` renvoient une erreur.

```
SW1(config)#vtp mode transparent unknown
```

---

## 6. Authentification

En version 3, le mot de passe peut être stocké en **hidden**.

```
SW1(config)#vtp password <PASSWORD> hidden
```

```
SW1#show vtp password
```

En hidden, le mot de passe n'apparaît que sous forme de **chaîne hexadécimale chiffrée**, jamais en clair, et on ne peut pas récupérer le mot de passe d'origine depuis le switch.
Pour ajouter un nouveau switch au domaine, collez cette chaîne en tant que **secret**.

```
SW4(config)#vtp password <encrypted-string> secret
```

> La désignation du Primary Server (`vtp primary`) demande aussi le mot de passe. Un mot de passe hidden protège même les changements de Primary Server.

---

## 7. VTP mode off

La version 3 prend en charge le mode `off`, qui **désactive complètement** VTP.

| Mode | Se synchronise | Relaie les advertisements |
| --- | --- | --- |
| Transparent | ❌ | ✅ continue à relayer |
| **Off** | ❌ | ❌ **ne relaie pas non plus** |

Désactiver globalement (par feature) :

```
SW1(config)#vtp mode off
SW1(config)#vtp mode off vlan
SW1(config)#vtp mode off mst
```

Désactiver par interface :

```
SW1(config)#interface GigabitEthernet0/1
SW1(config-if)#no vtp
```

Cette interface ne participe plus à VTP.

---

## 8. Compatibilité ascendante

- VTP version 3 est **compatible avec la version 2** et **incompatible avec la version 1.**
- Quand un switch en version 3 reçoit un advertisement de version 2, il relaie un advertisement compatible avec la version 2.
- La version 2 ne comprend pas le nouveau mécanisme de mot de passe : en environnement mixte, **retirez le mot de passe des switchs en version 3**.
- En cas de comportement étrange, vérifiez avec debug. (Dans le TP d'origine, un Catalyst 3750 avec un ancien IOS ne recevait pas du tout VTP.)

```
SW1#debug sw-vlan vtp events
SW1#no debug all
```

---

## Récapitulatif des commandes

| Commande | Description |
| --- | --- |
| `vtp domain <nom>` | Définit le nom de domaine (obligatoire en version 3) |
| `vtp version 3` | Passe en VTP version 3 |
| `vtp primary [vlan \| mst]` | Désigne le Primary Server (mode privilégié) |
| `vtp mode {server \| client \| transparent \| off} [vlan \| mst \| unknown]` | Mode VTP par feature |
| `vtp password <mot de passe> [hidden \| secret]` | Définit le mot de passe |
| `no vtp` (interface) | Désactive VTP sur cette interface |
| `show vtp status` | Affiche le mode, le Primary Server et la révision |
| `show vtp devices` | Affiche tous les switchs VTP du domaine |
| `show vtp password` | Affiche le mot de passe (chaîne chiffrée si hidden) |

---

## À retenir

- VTP version 3 **exige un nom de domaine**, et **seul le Primary Server** peut modifier les VLAN, ce qui évite l'accident « tous les VLAN effacés ».
- Il synchronise aussi les **VLAN étendus, Private VLAN, VLAN RSPAN et la configuration MST**.
- Les fonctions sont réparties en **features VLAN / MST / UNKNOWN**, chacune avec son propre mode et son Primary Server.
- Les **mots de passe hidden** et le **mode off** (global et par interface) ont été ajoutés.
- Il n'est compatible **qu'avec la version 2**.
