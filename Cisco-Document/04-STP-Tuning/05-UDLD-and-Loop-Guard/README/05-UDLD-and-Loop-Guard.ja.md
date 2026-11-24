# Spanning-Tree UDLD and Loop Guard

[한국어](../05-UDLD-and-Loop-Guard.md) | [English](05-UDLD-and-Loop-Guard.en.md) | **日本語** | [简体中文](05-UDLD-and-Loop-Guard.zh-CN.md) | [Español](05-UDLD-and-Loop-Guard.es.md) | [Français](05-UDLD-and-Loop-Guard.fr.md)

光ファイバーケーブルを使ったことがあれば、送信（transmit）と受信（receive）でコネクターが別々なことに気づいたかもしれません。このうち片方（送信か受信）だけが故障すると**単方向リンク障害（unidirectional link failure）** が起き、これが spanning-tree の**ループ**の原因になることがあります。この問題は 2 つのプロトコルで解決できます。

- **Loop Guard**
- **UDLD**（UniDirectional Link Detection）

![A fiber link uses separate transmit and receive strands](../images/udld-fiber.svg)

---

## 1. 単方向リンク障害が起きると？

まず、単方向リンク障害が起きると何が起こるのかを詳しく見てみましょう。

![Normal: SW3 receives BPDUs and blocks](../images/udld-normal.svg)

スイッチ間のリンクが光リンクだと考えてください。実際には送信と受信のコネクターが別々です。SW3 は SW2 から BPDU を受け取っているので、そのインターフェイスは **alternate port** になり、blocking 状態です。

![Unidirectional link failure → one-way loop](../images/udld-oneway.svg)

ところが問題が起きました。SW2 から SW3 へ向かう**送信コネクター**が原因不明で故障しました。そのため SW3 は SW2 から BPDU を受け取れなくなりましたが、SW2 にトラフィックを送ることはまだできます。

SW3 は alternate port で BPDU を受け取れなくなったので、そのポートを **forwarding** にします。これで緑の矢印のように**一方向のループ**ができてしまいます。

---

## 2. Loop Guard

単方向リンク障害を解決する方法の 1 つが **Loop Guard** です。スイッチがインターフェイスで BPDU を**送ってはいるのに受け取れない**とき、Loop Guard はそのインターフェイスを **loop-inconsistent** 状態にして、すべてのトラフィックをブロックします。

![Loop Guard: no BPDUs → loop-inconsistent](../images/udld-loopguard.svg)

実際のスイッチではどう見えるか確認してみましょう。同じ形のトポロジーを使います。

![Lab topology: SW1, SW2, SW3](../images/udld/01-topology.png)

> 原文では SW2 と SW3 の間に Fa0/16 を使っていますが、実習では **Gi0/1** でつないでいます。SW1 が root bridge で、SW3 の Gi0/1 が alternate port です。

3 台すべてで Loop Guard を有効にします。

![spanning-tree loopguard default on SW1, SW2, SW3](../images/udld/02-loopguard-default.png)

`spanning-tree loopguard default` コマンドは Loop Guard を**グローバル**で有効にします。光コネクターがないので実際の単方向リンク障害は作れませんが、SW2 の Gi0/1 で **BPDU Filter** を有効にすればまねできます。すると SW3 は alternate port で BPDU を受け取れなくなり、本来なら forwarding 状態に変わります。

![spanning-tree bpdufilter enable on SW2 Gi0/1](../images/udld/03-bpdufilter.png)

結果は次のとおりです。

![LOOPGUARD_BLOCK on SW3 Gi0/1](../images/udld/04-loopguard-block.png)

本来ならループが起きるところですが、幸い Loop Guard があります。コンソールに `LOOPGUARD_BLOCK` メッセージが出て、SW3 の Gi0/1 がブロックされます。問題解決です！

Loop Guard は必ずしもグローバルで有効にする必要はありません。インターフェイス単位でも次のように有効にできます。

![spanning-tree guard loop on SW3 interface](../images/udld/05-guard-loop.png)

---

## 3. UDLD

単方向リンク障害に対処するもう 1 つのプロトコルが **UDLD（UniDirectional Link Detection）** です。spanning-tree の機能ではありませんが、ループを防ぐのに役立ちます。

簡単に言うと、UDLD は **keepalive のように動作する L2 プロトコル**です。hello メッセージを送り、受け取れていれば問題ありません。ところが hello を送っているのに**受け取れなくなった**ら何かがおかしいので、インターフェイスをブロックします。

同じトポロジーで、今度は UDLD を設定します。先に Loop Guard を削除するのを忘れないでください。

UDLD の設定方法はいくつかあります。`udld` コマンドでグローバルに有効にすることもできますが、この方法では**光ポートだけ**で UDLD が有効になります。

![udld ? on SW1](../images/udld/06-udld-options.png)

UDLD には 2 つのモードがあります。

| モード | 動作 |
| --- | --- |
| Normal（既定） | 問題が起きるとポートを **undetermined** と表示するだけで、インターフェイスは落としません。知らせるだけで、ループは止められません。 |
| Aggressive | 隣接装置との接続が切れると、UDLD フレームを **1 秒に 1 回、8 秒間**送ります。それでも隣接装置が応答しなければ、インターフェイスを **err-disable** にします。 |

**Aggressive** のほうがよい解決策です。スイッチ 2 台で UDLD を確認します。

![Lab topology: SW2, SW3](../images/udld/07-topology-udld.png)

問題が起きるとインターフェイスが落ちるのを見るため、SW2 と SW3 の Gi0/1 で **aggressive** モードを使います。リアルタイムで見るために debug も有効にします。

![udld port aggressive on SW2 and SW3](../images/udld/08-udld-aggressive.png)

正常なときは隣接装置を見つけ、リンクが双方向かを確認します（`Found my own ID pair in 2way conn list`）。

![UDLD debug: link is bidirectional](../images/udld/09-udld-debug.png)

難しいのは単方向リンク障害をまねることです。Loop Guard は BPDU をもとにしているので簡単でした。UDLD は独自の L2 プロトコルなので、SW3 で **MAC access-list** を使い、SW2 が送るフレーム（SW2 の MAC `502c.5b00.0300`）をブロックして UDLD のトラフィックを遮断します。

![mac access-list extended UDLD-FILTER on SW3](../images/udld/10-mac-filter.png)

> 原文は `502c.5b00.0300` を UDLD 専用の MAC アドレスと説明していますが、実習のトポロジーではこのアドレスは **SW2 の MAC** です。実習では SW3 で SW2 が送るフレームを止める方法で UDLD を遮断しました。

片側で UDLD フレームをフィルターするという工夫した方法です。こうすると、単方向リンク障害が起きたと判断します。結果は次のとおりです。

![UDLD debug: phase udld_detection](../images/udld/11-udld-detection.png)

debug メッセージが大量に流れ、出力では **`phase udld_detection`** の段階に移ったことがわかります。原文によると、最終的にポートは **err-disable** 状態になります。次のコマンドで確認できます。

```
SW2#show udld
SW2#show interfaces status err-disabled
```

---

## 4. Loop Guard vs UDLD

Loop Guard と UDLD は、どちらも**単方向リンク障害**という同じ問題を解決します。重なる部分もありますが、違いもあります。

| 項目 | Loop Guard | UDLD |
| --- | --- | --- |
| 設定 | グローバル / ポートごと | グローバル（光ポート用）/ ポートごと |
| VLAN ごとの動作 | はい | いいえ、ポート単位 |
| 自動復旧 | はい | はい – err-disable timeout の設定が必要 |
| 単方向リンク障害による STP 障害の防止 | はい – すべての root・alternate ポートで有効にする必要あり | はい – すべてのインターフェイスで有効にする必要あり |
| BPDU が送られないことによる STP 障害の防止 | はい | いいえ |
| 誤配線（miswiring）の防止 | いいえ | はい |

---

## まとめ

- 光リンクは送信と受信の芯線が別々なので、片方だけ故障すると**単方向リンク障害**が起きます。BPDU を受け取れなくなった alternate port が forwarding になり、**一方向のループ**ができることがあります。
- **Loop Guard** は BPDU を送っているのに受け取れないポートを **loop-inconsistent** でブロックします。グローバル `spanning-tree loopguard default`、インターフェイス `spanning-tree guard loop`。ブロックすると `LOOPGUARD_BLOCK` メッセージが出ます。
- **UDLD** は hello をやり取りする L2 の keepalive プロトコルです。グローバルの `udld` コマンドは光ポートだけに適用され、ポートでは `udld port [aggressive]` で有効にします。
- UDLD の **normal** は通知（undetermined）だけ、**aggressive** は 8 秒間確認したあと **err-disable** でポートをブロックします。
- Loop Guard は VLAN ごとに動作し、BPDU が来ない障害まで防ぎます。UDLD はポート単位で動作し、**誤配線**まで検出します。両方を一緒に使うと互いを補えます。
