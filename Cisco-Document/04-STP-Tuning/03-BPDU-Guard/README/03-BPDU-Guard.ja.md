# Spanning-Tree BPDU Guard

[한국어](../03-BPDU-Guard.md) | [English](03-BPDU-Guard.en.md) | **日本語** | [简体中文](03-BPDU-Guard.zh-CN.md) | [Español](03-BPDU-Guard.es.md) | [Français](03-BPDU-Guard.fr.md)

**BPDU Guard** は spanning-tree のトポロジーを守る機能の 1 つです。BPDU Guard を有効にしたインターフェイスに **BPDU が入ってくると、そのインターフェイスを err-disable 状態**にします。なぜ必要なのか、例で見てみましょう。

## BPDU Guard の設定方法

| 場所 | コマンド | 適用対象 |
| --- | --- | --- |
| インターフェイス | `spanning-tree bpduguard enable` | そのインターフェイス |
| グローバル | `spanning-tree portfast bpduguard default` | PortFast が有効なすべてのインターフェイス |

---

## 1. なぜ BPDU Guard が必要なのか？

![An attacker sends superior BPDUs](../images/bpduguard-attack.svg)

上のトポロジーでは spanning-tree が正しく動作しています。既定では spanning-tree は**すべてのインターフェイスで BPDU を送受信します**。この例では SW2 の fa0/2 にコンピューターがつながっています。好奇心の強い人や悪意のある人が、**よりよい（superior）bridge ID を持つ BPDU** を生成するツールを動かすかもしれません。するとスイッチたちは root bridge が SW2 の先にあると思い込み、spanning-tree を**再計算**します。よいことではありませんよね？ 何がまずいのか見てみましょう。

![Man-in-the-middle: the attacker becomes the root](../images/bpduguard-mitm.svg)

誰にも気づかれずに**中間者攻撃（man-in-the-middle）** をすることさえできます。攻撃者がコンピューターを 2 台のスイッチにつないで root bridge になれば、スイッチ間を行き来するトラフィックはすべて攻撃者のコンピューターを通ります。あとは Wireshark を動かしておくだけです。

BPDU Guard を使えば、これを防げます。BPDU が入ってきた瞬間にインターフェイスを止めてしまうからです。

![BPDU Guard drops the attacker](../images/bpduguard-block.svg)

BPDU Guard は、インターフェイスで BPDU を受け取るとそのインターフェイスを **err-disable** モードにします。設定方法を見てみましょう。

---

## 2. インターフェイスに設定する

次のトポロジーを使います。

![Lab topology: SW2, SW3](../images/bpdu-guard/01-topology.png)

スイッチ 2 台で BPDU Guard を確認します。SW2 の Gi0/0 が SW3 から BPDU を受け取ると err-disable モードになるように設定します。

![spanning-tree bpduguard enable on Gi0/0](../images/bpdu-guard/02-bpduguard.png)

インターフェイスで有効にする方法です。実際にはスイッチ間のリンクでこうすることはありません。**コンピューターにつながる access モードのインターフェイス**に設定します。

![BLOCK_BPDUGUARD and err-disable](../images/bpdu-guard/03-err-disable.png)

予想どおりインターフェイスが落ちました。Gi0/0 で BPDU を受け取ると `BLOCK_BPDUGUARD` メッセージとともに **err-disable** 状態になり、line protocol も down になります。BPDU Guard を削除し、インターフェイスを `shutdown` / `no shutdown` して復旧させます。

![no spanning-tree bpduguard, shutdown, no shutdown](../images/bpdu-guard/04-remove.png)

---

## 3. グローバルに設定する

インターフェイスごとに有効にする代わりに、**グローバル**に有効にすることもできます。実習機器（IOSv）にはこのコマンドがなかったため、この部分は **Packet Tracer** で行いました。

```
SW2(config)#spanning-tree portfast bpduguard default
```

このコマンドは **PortFast が有効なすべてのインターフェイス**で BPDU Guard を有効にします。そのため、通常は PortFast もグローバルで一緒に有効にします。

```
SW2(config)#spanning-tree portfast default
```

設定を確認するのに便利なコマンドです。

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

**Portfast Default** と **PortFast BPDU Guard Default** がグローバルで有効になっていることがわかります。

---

## まとめ

- 既定ではすべてのインターフェイスが BPDU を送受信するので、ホストのポートから **superior BPDU** を送ると spanning-tree が再計算され、攻撃者が root bridge になって**中間者攻撃**ができてしまいます。
- **BPDU Guard** は BPDU を受け取ったインターフェイスを **err-disable** にします（`BLOCK_BPDUGUARD`、`ERR_DISABLE` メッセージ）。
- インターフェイス: `spanning-tree bpduguard enable` / グローバル: `spanning-tree portfast bpduguard default`（PortFast ポート全体）。
- スイッチ間のリンクではなく、**コンピューターにつながる access ポート**で使います。err-disable になったポートは、原因を取り除いてから `shutdown` / `no shutdown` で復旧させます。
- `show spanning-tree summary` でグローバルの PortFast と BPDU Guard の設定を確認します。
