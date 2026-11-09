# How to Troubleshoot VLANs and Trunks

[한국어](../08-How-to-Troubleshoot-VLANs-and-Trunks.md) | [English](08-How-to-Troubleshoot-VLANs-and-Trunks.en.md) | [日本語](08-How-to-Troubleshoot-VLANs-and-Trunks.ja.md) | [简体中文](08-How-to-Troubleshoot-VLANs-and-Trunks.zh-CN.md) | [Español](08-How-to-Troubleshoot-VLANs-and-Trunks.es.md) | **Français**

Si vous avez déjà vérifié la configuration des interfaces (speed / duplex, erreurs, etc.) et que les machines ne communiquent toujours pas, le problème vient peut-être des **VLAN ou des trunks**. Voici 5 situations courantes, avec les captures du TP.

## L'ordre de vérification en un coup d'œil

| Étape | Ce qu'il faut vérifier | Commande |
| --- | --- | --- |
| 1 | Les interfaces sont-elles up / up ? | `show ip interface brief`, `show interfaces <if>` |
| 2 | Les deux ports sont-ils dans le **même VLAN** ? | `show vlan` |
| 3 | Le port est-il dans le **bon mode (access / trunk)** ? | `show interfaces <if> switchport` |
| 4 | Le port security bloque-t-il quelque chose ? | `show port-security` |
| 5 | Une **VACL** bloque-t-elle le trafic dans le VLAN ? | `show vlan filter`, `show vlan access-map` |
| 6 | Les deux extrémités du trunk utilisent-elles la **même encapsulation** ? | `show interfaces <if> switchport` |
| 7 | Le trunk **autorise**-t-il ce VLAN ? | `show interfaces <if> trunk` |

---

## 1. Problèmes d'affectation de VLAN

![Topologie à un switch](../images/troubleshooting/topology-single-switch.png)

VPC11 (192.168.1.1/24) n'arrive pas à pinguer VPC12 (192.168.1.2/24). Les machines vont bien ; le problème vient du switch.

![Le ping échoue](../images/troubleshooting/c1-ping-fail.png)

Vérifiez d'abord les interfaces.

![show ip int brief](../images/troubleshooting/c1-ip-int-brief.png)

Les deux sont up / up. Vérifiez maintenant l'affectation des VLAN.

![show vlan](../images/troubleshooting/c1-show-vlan.png)

`show vlan` montre **Gi0/0 dans le VLAN 1 et Gi0/1 dans le VLAN 2**. Même sous-réseau, mais **VLAN différents** : elles ne peuvent pas communiquer. Déplacez Gi0/1 dans le VLAN 1.

![Passage dans le VLAN 1](../images/troubleshooting/c1-fix-access-vlan.png)

![Le ping fonctionne](../images/troubleshooting/c1-ping-ok.png)

> **Leçon** : si deux machines du même sous-réseau ne communiquent pas, vérifiez d'abord avec `show vlan` que **les deux ports sont dans le même VLAN**.

---

## 2. Problèmes de mode switchport

Même topologie, autre problème. Les deux interfaces sont up / up. Regardez l'affectation des VLAN.

![show vlan](../images/troubleshooting/c2-show-vlan.png)

**Gi0/1 est dans le VLAN 10**, mais **Gi0/0 n'apparaît nulle part.** Deux causes possibles :
- L'interface est en panne → non, elle est up / up
- L'interface est un **trunk** et non un port d'accès (`show vlan` n'affiche que les ports access)

![Informations switchport](../images/troubleshooting/c2-switchport-trunk.png)

Gi0/0 est en **mode trunk** avec le VLAN natif 1. Le PC n'ajoute pas d'étiquette 802.1Q, donc le trafic de VPC11 arrive dans le **VLAN natif 1** et ne peut pas joindre VPC12 dans le VLAN 10. Passez-le en mode access et placez-le dans le VLAN 10.

![Passage en mode access](../images/troubleshooting/c2-fix-access-mode.png)

![show vlan id 10](../images/troubleshooting/c2-show-vlan-id-10.png)

Les deux ports sont maintenant dans le VLAN 10. Vérifiez aussi le mode opérationnel.

![Mode opérationnel](../images/troubleshooting/c2-operational-mode.png)

![Le ping fonctionne](../images/troubleshooting/c2-ping-ok.png)

> **Leçon** : assurez-vous que le port est dans le **bon mode switchport (access / trunk)**. Si un port n'apparaît pas dans `show vlan`, c'est probablement un trunk.

---

## 3. Problèmes de VACL (VLAN access-list)

Mêmes PC, même switch, encore un autre problème. Vérifions point par point.

![show ip int brief](../images/troubleshooting/c3-ip-int-brief.png)

Interfaces up / up. Affectation des VLAN ?

![show vlan](../images/troubleshooting/c3-show-vlan.png)

Les deux ports sont dans le **VLAN 10**. Le port security, alors ?

![port-security](../images/troubleshooting/c3-port-security.png)

Pas de port security non plus. Interfaces correctes, même VLAN, pas de port security… si le trafic est toujours bloqué, soupçonnez une **VACL**. Une VACL peut autoriser ou bloquer le trafic **à l'intérieur d'un même VLAN**.

![show vlan filter](../images/troubleshooting/c3-vlan-filter.png)

Un VLAN access-map nommé `BLOCKSTUFF` est appliqué au VLAN 10.

![show vlan access-map](../images/troubleshooting/c3-access-map.png)

![show access-lists](../images/troubleshooting/c3-access-list.png)

- L'action de l'access-map est **drop**.
- La condition est l'access-list 1, et son `permit 192.168.1.0 0.0.0.255` ne veut pas dire « autoriser » mais **« correspond à cette plage »**.
- Les deux PC sont dans cette plage : ils correspondent → **rejetés**.

Changez l'action en **forward**.

![action forward](../images/troubleshooting/c3-fix-action-forward.png)

![Le ping fonctionne](../images/troubleshooting/c3-ping-ok.png)

> **Leçon** : si tout le reste semble correct mais que les machines d'un même VLAN ne communiquent pas, cherchez une **VACL**. Un `permit` dans l'ACL d'une VACL signifie « correspondance » ; c'est l'**action** de l'access-map qui décide de transmettre ou de rejeter.

---

## 4. Problèmes d'encapsulation du trunk

Jusqu'ici, les problèmes étaient internes à un seul switch. Quand le trafic doit traverser plusieurs switchs, vérifiez aussi les **trunks**.

![Topologie à deux switchs](../images/troubleshooting/topology-two-switches.png)

Chaque switch a un PC dans le VLAN 10, mais ils ne se joignent pas. Commencez par le port de SW1 relié au PC.

![SW1 int g0/0](../images/troubleshooting/c4-sw1-int-g0-0.png)

![SW1 g0/0 switchport](../images/troubleshooting/c4-sw1-g0-0-switchport.png)

![SW1 port-security](../images/troubleshooting/c4-sw1-port-security.png)

Up, access, **VLAN 10**, pas de port security. Vérifiez la même chose sur SW2.

![SW2 int g0/0](../images/troubleshooting/c4-sw2-int-g0-0.png)

![SW2 g0/0 switchport](../images/troubleshooting/c4-sw2-g0-0-switchport.png)

![SW2 port-security](../images/troubleshooting/c4-sw2-port-security.png)

Les ports vers les PC sont corrects des deux côtés. Deux options maintenant :
- Brancher un autre PC du VLAN 10 sur le même switch et vérifier si la communication fonctionne au sein du switch
- Vérifier le **lien entre SW1 et SW2** ← là où beaucoup de choses peuvent mal tourner

![SW1 int g0/1](../images/troubleshooting/c4-sw1-int-g0-1.png)

![SW2 int g0/1](../images/troubleshooting/c4-sw2-int-g0-1.png)

Les interfaces sont correctes. Regardez les informations switchport.

![SW1 g0/1 ISL](../images/troubleshooting/c4-sw1-g0-1-isl.png)

![SW2 g0/1 dot1q](../images/troubleshooting/c4-sw2-g0-1-dot1q.png)

Les deux sont des trunks, mais **SW1 utilise ISL et SW2 802.1Q**. Avec des encapsulations différentes, aucun des deux ne comprend les trames de l'autre. Passez SW1 en 802.1Q.

![Changement d'encapsulation](../images/troubleshooting/c4-fix-encapsulation.png)

![Le ping fonctionne](../images/troubleshooting/c4-ping-ok.png)

> **Leçon** : les deux extrémités d'un trunk doivent utiliser la **même encapsulation**.
> De plus, selon le modèle de switch, le mode par défaut peut être `dynamic auto`, et **si les deux extrémités sont en `dynamic auto`, le lien devient access**. Le plus sûr est de configurer explicitement les trunks avec `switchport mode trunk`. ([05. DTP](../05-Dynamic-Trunking-Protocol.md))

---

## 5. Problèmes de VLAN autorisés sur le trunk

Même topologie. On suppose que ces points ont déjà été vérifiés et sont corrects :
- Interfaces (speed / duplex)
- Port security
- Configuration switchport (affectation de VLAN, mode access)

![Le ping échoue](../images/troubleshooting/c5-ping-fail.png)

Vérifiez le trunk entre les switchs.

![SW1 g0/1 switchport](../images/troubleshooting/c5-sw1-g0-1-switchport.png)

![SW2 g0/1 switchport](../images/troubleshooting/c5-sw2-g0-1-switchport.png)

Les deux sont des trunks, tous deux en 802.1Q. Jusqu'ici, tout va bien. Mais un trunk opérationnel **ne signifie pas que tous les VLAN peuvent le traverser.**

![VLAN autorisés sur SW1](../images/troubleshooting/c5-sw1-trunk-allowed.png)

![VLAN autorisés sur SW2](../images/troubleshooting/c5-sw2-trunk-allowed.png)

`Vlans allowed on trunk` ne contient que **20**. Le VLAN 10 ne peut pas traverser ce trunk. Autorisez tous les VLAN.

![SW1 allowed vlan all](../images/troubleshooting/c5-sw1-fix-allowed.png)

![SW2 allowed vlan all](../images/troubleshooting/c5-sw2-fix-allowed.png)

![Le ping fonctionne](../images/troubleshooting/c5-ping-ok.png)

> **Leçon** : vérifiez toujours que le trunk **autorise tous les VLAN** nécessaires.
> En production, on n'autorise souvent que les VLAN utiles au lieu de `all`. Pour **ajouter** un VLAN à la liste existante, utilisez `switchport trunk allowed vlan add 10`. Taper `allowed vlan 10` sans `add` **écrase** la liste et peut couper d'autres VLAN.

---

## Récapitulatif des corrections

| Cas | Cause | Correction |
| --- | --- | --- |
| 1. Affectation de VLAN | Les deux ports sont dans des VLAN différents | `switchport access vlan 1` |
| 2. Mode switchport | Le port du PC est un trunk | `switchport mode access` → `switchport access vlan 10` |
| 3. VACL | L'action de l'access-map est drop | `vlan access-map BLOCKSTUFF 10` → `action forward` |
| 4. Encapsulation du trunk | ISL ↔ 802.1Q incohérents | `switchport trunk encapsulation dot1q` |
| 5. VLAN autorisés | Le trunk n'autorise que le VLAN 20 | `switchport trunk allowed vlan all` |

## À retenir

- Procédez **de bas en haut** : interface → affectation de VLAN → mode switchport → port security → VACL → trunk.
- Si un port n'apparaît pas dans `show vlan`, c'est peut-être un **trunk**.
- Si des machines d'un même VLAN sont toujours bloquées, soupçonnez une **VACL**.
- Pour les trunks, vérifiez le **mode et l'encapsulation des deux côtés** ainsi que la **liste des VLAN autorisés**.
