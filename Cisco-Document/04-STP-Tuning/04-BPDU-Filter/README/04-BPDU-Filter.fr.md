# Spanning-Tree BPDU Filter

[한국어](../04-BPDU-Filter.md) | [English](04-BPDU-Filter.en.md) | [日本語](04-BPDU-Filter.ja.md) | [简体中文](04-BPDU-Filter.zh-CN.md) | [Español](04-BPDU-Filter.es.md) | **Français**

Le **BPDU Filter** fonctionne un peu comme [BPDU Guard](../../03-BPDU-Guard/README/03-BPDU-Guard.fr.md) : il permet de bloquer les BPDU malveillantes. La différence est que BPDU Guard met en **err-disable** l'interface qui reçoit la BPDU, alors que BPDU Filter se contente de la **filtrer**. Dans cette leçon, nous examinons en détail le fonctionnement de BPDU Filter.

## Configuration globale ou par interface

BPDU Filter peut être configuré globalement ou sur l'interface, et le comportement diffère.

| Où | Commande | Comportement |
| --- | --- | --- |
| Global | `spanning-tree portfast bpdufilter default` | Les **interfaces avec PortFast activé** n'envoient ni ne reçoivent de BPDU. Mais si une BPDU arrive sur l'une d'elles, elle **perd son statut PortFast, BPDU Filter est désactivé** et elle se comporte comme une interface normale. |
| Interface | `spanning-tree bpdufilter enable` | Les BPDU entrantes sont **ignorées** et **aucune BPDU n'est envoyée**. Cela revient à **désactiver spanning tree** sur cette interface. |

Soyez prudent en activant BPDU Filter sur des interfaces. Vous pouvez l'utiliser sur des interfaces en mode access connectées à des ordinateurs, mais **ne le configurez jamais sur des interfaces connectées à d'autres commutateurs** ; sinon, vous risquez une boucle.

---

## 1. Topologie du labo

Nous utilisons cette topologie pour montrer BPDU Filter.

![Lab topology: SW2, SW3](../images/bpdu-filter/01-topology.png)

---

## 2. Configuration sur une interface

Activons PortFast sur l'interface de SW2, puis BPDU Filter.

![spanning-tree portfast trunk, spanning-tree bpdufilter enable](../images/bpdu-filter/02-bpdufilter.png)

Le lien du labo est un trunk, PortFast a donc été activé avec `spanning-tree portfast trunk`. SW2 cesse d'envoyer des BPDU sur cette interface et ignore tout ce qu'il reçoit. Activons un debug pour voir ce qu'il fait.

![debug spanning-tree bpdu on SW2](../images/bpdu-filter/03-debug.png)

Avec le debug des BPDU activé, **vous ne verrez rien**, car SW2 n'envoie ni ne traite plus de BPDU. Si vous le souhaitez, activez aussi le debug des BPDU sur SW3 : vous verrez qu'il n'en reçoit aucune de SW2. Supprimons la commande d'interface.

![no spanning-tree bpdufilter enable](../images/bpdu-filter/04-remove.png)

---

## 3. Configuration globale

Activons maintenant BPDU Filter globalement.

```
SW2(config)#spanning-tree portfast bpdufilter default
```

> Dans le PDF, la capture de cette étape est le même écran que la capture précédente de `no spanning-tree bpdufilter enable` ; la commande globale est donc présentée en bloc de code.

Cette commande active BPDU Filter sur **toutes les interfaces où PortFast est activé**.

C'est tout. Il est toutefois sans doute **préférable d'utiliser BPDU Guard**. Si vous n'attendez pas de BPDU sur une interface, mieux vaut **être averti** par l'err-disable que de les filtrer en silence sans voir ce qui se passe.

---

## Résumé

- **BPDU Filter** **filtre** les BPDU sans err-disable ; BPDU Guard, lui, met le port en err-disable.
- **Global** (`spanning-tree portfast bpdufilter default`) : les ports PortFast n'envoient ni ne reçoivent de BPDU, mais en recevoir une leur retire PortFast et BPDU Filter, et ils redeviennent des ports normaux.
- **Interface** (`spanning-tree bpdufilter enable`) : ignore les BPDU et n'en envoie aucune, ce qui revient à **désactiver spanning tree** sur ce port.
- L'utiliser sur des ports connectés à d'autres commutateurs peut créer des **boucles**. Réservez-le aux ports d'hôte et préférez **BPDU Guard**, qui vous avertit.
