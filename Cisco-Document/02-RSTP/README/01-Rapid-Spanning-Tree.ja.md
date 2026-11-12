# Rapid Spanning-Tree (RSTP)

[한국어](../01-Rapid-Spanning-Tree.md) | [English](01-Rapid-Spanning-Tree.en.md) | **日本語** | [简体中文](01-Rapid-Spanning-Tree.zh-CN.md) | [Español](01-Rapid-Spanning-Tree.es.md) | [Français](01-Rapid-Spanning-Tree.fr.md)

最近のネットワークではルーティングがますます増えています。OSPF や EIGRP などのルーティングプロトコルは、ネットワークの変化に Spanning Tree よりはるかに速く対応します。この速さについていくために作られたのが **Rapid Spanning Tree (RSTP、IEEE 802.1w)** です。

RSTP は従来の Spanning Tree の**革命ではなく進化**です。速くするために内部動作をいくつか変えただけで、**設定方法は同じ**です。以下では従来の Spanning Tree(802.1D)を **Classic Spanning Tree** と呼びます。

## 違いの一覧

| 項目 | Classic STP (802.1D) | RSTP (802.1w) |
| --- | --- | --- |
| ポート状態 | Blocking / Listening / Learning / Forwarding | **Discarding** / Learning / Forwarding |
| ポートの役割 | Root、Designated、Alternate(Blocked) | Root、Designated、Alternate、**Backup** |
| BPDU | Flags は 2 ビットのみ使用(TC、TCA) | **Version 2 BPDU**、Flags をすべて使用 |
| BPDU の生成 | Root のみ生成、他は中継 | **すべてのスイッチが hello time(2 秒)ごと**に生成 |
| 障害検知 | Max Age 20 秒 | BPDU を **3 回取りこぼす**と検知(キープアライブ) |
| Inferior BPDU | 無視 | **受け入れる**(BackboneFast に近い) |
| Forwarding への移行 | タイマー方式(Listening → Learning、約 30 秒) | **Proposal / Agreement のネゴシエーション(Sync)** |
| PortFast | PortFast | **Edge port** |
| UplinkFast | 手動で設定 | **標準で内蔵** |
| Topology Change | TCN を Root まで送信 → Root が通知 | **検知したスイッチがすぐにフラッディング** |

---

## 1. ポート状態: 4 つ → 3 つ

![Port states](../images/rstp-port-states.svg)

RSTP のポート状態は **Discarding、Learning、Forwarding** の 3 つだけです。**Discarding** は Classic の Blocking と Listening をまとめた状態です。

| Classic Spanning Tree | Rapid Spanning Tree | トポロジーで有効? | MAC アドレスを学習? |
| --- | --- | --- | --- |
| Blocking | Discarding | No | No |
| Listening | Discarding | Yes | No |
| Learning | Learning | Yes | Yes |
| Forwarding | Forwarding | Yes | Yes |

---

## 2. ポートの役割

![Port roles](../images/rstp-port-roles.svg)

- **Root Bridge**: Bridge ID(Priority + MAC)が最も良いスイッチ。ここでは SW1(MAC AAA)。
- **Root port**: Non-root スイッチで Root までのコストが最も低いポート。
- **Designated port**: セグメントごとに 1 つだけ。**最も良い BPDU を送れる**ポートがなります。
- **Alternate port**: SW3 の Fa0/16 は SW2 からより良い BPDU を受け取るのでブロックされます。Root port の予備経路です。

ここまでは Classic Spanning Tree と同じです。RSTP で新しく追加された役割が **Backup port** です。

### Backup port

![Backup port](../images/rstp-backup-port.svg)

SW2 と SW3 の間に **ハブ** を入れました。ハブがなければ SW2 の Fa0/16、Fa0/17 はどちらも Designated port になります。

しかしハブのせいで、SW2 の Fa0/16 と Fa0/17 は**同じコリジョンドメイン**に入りました。SW2 は自分が送った BPDU を Fa0/16 と Fa0/17 の両方で受け取るので、同じセグメントに 2 本つながっていると分かります。

- Fa0/16 → **Designated port**
- Fa0/17 → Fa0/16 の **Backup port**

> 実際のネットワークで Backup port を見ることはほとんどありません。ハブを外せば、再び両方とも Designated port になります。

---

## 3. BPDU(Version 2)

![RSTP BPDU](../images/rstp-bpdu-flags.svg)

Classic Spanning Tree は Flags フィールドの **2 ビットだけ**を使っていました。
- Topology Change
- Topology Change Acknowledgement

RSTP は **Flags フィールドをすべて**使います。特に **Port Role** フィールドで、BPDU を送ったポートの役割を伝えます。
- Unknown
- Alternate / Backup port
- Root port
- Designated port

この新しい BPDU を **Version 2 BPDU** と呼びます。Classic Spanning Tree だけを使うスイッチはこの BPDU を破棄します。

---

## 4. その他の変更点

### BPDU はすべてのスイッチが hello time ごとに送る
Classic では **Root Bridge だけ** が BPDU を生成し、Non-root スイッチは Root port で受け取った BPDU を中継するだけでした。RSTP では **すべてのスイッチが 2 秒(hello time)ごと** に自分で BPDU を生成します。hello time は変更できます。

### BPDU = キープアライブ
Classic は BPDU を破棄するまで **Max Age(20 秒)** 待っていました。RSTP は OSPF・EIGRP の hello のように BPDU を **キープアライブ** として使います。隣接スイッチの BPDU を **3 回続けて取りこぼす** と接続が切れたと判断し、MAC アドレスをすぐに削除します。

### Inferior BPDU を受け入れる
Classic は inferior BPDU を無視しますが、RSTP は受け入れます。Classic Spanning Tree の **BackboneFast** とほぼ同じ機能です。

### タイマーではなくネゴシエーション
Classic は Forwarding の前に Listening、Learning を通る必要があり、デフォルトのタイマーで **30 秒** かかりました。RSTP はタイマーの代わりに **ネゴシエーション** で Forwarding するかを決めます。これが RSTP で最も重要な変化、**高速な収束(コンバージェンス)** です。

### PortFast = Edge port
Classic で PortFast を有効にすると Listening・Learning を飛ばしてすぐ Forwarding になり、ポートの up/down で Topology Change も発生しません。RSTP でも PortFast はそのまま使いますが、**Edge port** と呼びます。

### Link type
RSTP がポートを素早く Forwarding にできるのは **Edge port** と **Point-to-point** リンクだけです。

| Link type | Duplex | 説明 |
| --- | --- | --- |
| Point-to-point | Full duplex | スイッチ同士をつなぐ通常のリンク |
| Shared | Half duplex | ハブが間に入っているリンク |

---

## 5. Sync: Proposal / Agreement

![RSTP sync overview](../images/rstp-sync-overview.svg)

SW1 が Root Bridge で、SW2、SW3、SW4 は Non-root です。

1. SW1–SW2 のリンクが上がると、両方のポートは**ブロックされた状態**で始まります。SW2 が SW1 の BPDU を受け取ると、**Sync** というネゴシエーションが始まります。
2. SW2 は Root の BPDU を受け取るとすぐに、**non-edge の Designated port をすべてブロック**します(non-edge = 他のスイッチへのポート、edge = PortFast のポート)。すると SW1–SW2 のリンクが **Forwarding** になります。
3. SW2 は続けて SW3、SW4 とも Sync を行い、そちらのリンクも素早く Forwarding にします。

実際にどのメッセージがやり取りされるかを順番に見ると、次のようになります。

![Proposal and agreement](../images/rstp-proposal-agreement.svg)

| ステップ | 動作 |
| --- | --- |
| 1. Proposal | SW1 が BPDU Flags の **Proposal ビット**を立てて送信。SW2 は SW1 が Root だと分かる |
| 2. Sync | SW2 が **non-edge ポートをすべてブロック**し、SW3、SW4 側の同期を準備 |
| 3. Agreement | SW2 が **Agreement** を送信(Proposal BPDU のコピーで Proposal OFF、Agreement ON)。SW2 Fa0/14 が Forwarding |
| 4. Forwarding | Agreement を受け取った SW1 が Fa0/14 を**すぐに Forwarding** に |
| 5. Proposal | SW2 が Fa0/16(SW3)、Fa0/19(SW4)へ同じ Proposal を送信 |
| 6. Agreement | SW3、SW4 は他にポートがないので、すぐに Agreement を返す |
| 7. Forwarding | SW2 が Fa0/16、Fa0/19 を Forwarding にして完了 |

> メッセージが数回行き来するだけなので、Classic のタイマー方式(Listening → Learning → Forwarding)より**ずっと速い**です。

---

## 6. UplinkFast は標準装備

Classic では **UplinkFast** を手動で有効にする必要がありました。RSTP には標準で入っています。Root port を失うと、**Alternate port をすぐに Forwarding** にします。

Classic の UplinkFast は、すべてのスイッチの MAC アドレステーブルを更新するためにマルチキャストフレームを送る必要がありました。RSTP は Topology Change の仕組み自体が違うので、これは不要です。

---

## 7. Topology Change

![Topology change](../images/rstp-topology-change.svg)

Classic では**リンク障害も Topology Change** でした。RSTP ではリンク障害を Topology Change とは見なしません。**non-edge ポート(他のスイッチへのポート)が Forwarding になったときだけ** Topology Change です。

Topology Change を**検知したスイッチ**は:
- すべての non-edge Designated port と Root port について **topology change while タイマー**を開始します。値は **hello time の 2 倍**です。
- これらのポートで学習した **MAC アドレスを削除**します。
- タイマーが動いている間、これらのポートから送る BPDU に **TC ビット**を立てます。Root port からも BPDU を送ります。

TC ビットが立った BPDU を**受け取った隣接スイッチ**は:
- BPDU を受け取ったポートを**除くすべてのポート**の MAC アドレスを削除します。
- 自分でも topology change while タイマーを開始し、すべての Designated port と Root port へ TC ビットを立てた BPDU を送ります。

> Classic のように Root Bridge まで上げてから下ろすのではなく、**ネットワーク全体へすぐにフラッディング**されます。

---

## 8. Classic Spanning Tree との互換性

![Compatibility](../images/rstp-compatibility.svg)

RSTP と Classic Spanning Tree は**互換性があります**。ただし、RSTP スイッチが Classic スイッチと接する区間では、**RSTP を速くしている機能が働きません**。

図では SW1–SW2 間は RSTP で動きますが、SW2–SW3 間は **Classic Spanning Tree に戻ります**。

---

## まとめ

- RSTP(802.1w)は Classic STP の**進化**。設定方法は同じで、内部動作が速くなった。
- ポート状態は **Discarding / Learning / Forwarding** の 3 つ。新しい役割は **Backup port**。
- **Version 2 BPDU**: Flags をすべて使い、Port Role を含む。すべてのスイッチが 2 秒ごとに送り、3 回取りこぼすと障害と判断。
- タイマーの代わりに **Proposal → Sync → Agreement** のネゴシエーションで収束。高速な移行は **Edge port** と **Point-to-point** リンクだけ。
- **UplinkFast は標準内蔵**。Topology Change は **non-edge ポートが Forwarding になったときだけ**で、**フラッディング**で素早く伝わる。
- Classic STP と互換性はあるが、接している区間は Classic として動作。
