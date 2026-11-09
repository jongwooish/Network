# How to Troubleshoot VLANs and Trunks

[한국어](../08-How-to-Troubleshoot-VLANs-and-Trunks.md) | [English](08-How-to-Troubleshoot-VLANs-and-Trunks.en.md) | **日本語** | [简体中文](08-How-to-Troubleshoot-VLANs-and-Trunks.zh-CN.md) | [Español](08-How-to-Troubleshoot-VLANs-and-Trunks.es.md) | [Français](08-How-to-Troubleshoot-VLANs-and-Trunks.fr.md)

インターフェースの設定(speed / duplex、エラーなど)を確認しても通信できない場合、問題は **VLAN や Trunk** 側にあるかもしれません。よくある5つの状況を、実習画面とあわせてまとめます。

## 確認の順番

| 順番 | 確認すること | コマンド |
| --- | --- | --- |
| 1 | インターフェースは up / up か | `show ip interface brief`、`show interfaces <if>` |
| 2 | 2つのポートは**同じ VLAN** か | `show vlan` |
| 3 | ポートは **access / trunk の正しいモード** か | `show interfaces <if> switchport` |
| 4 | Port Security が止めていないか | `show port-security` |
| 5 | **VACL** が VLAN 内のトラフィックを止めていないか | `show vlan filter`、`show vlan access-map` |
| 6 | Trunk 両端の**カプセル化は同じ**か | `show interfaces <if> switchport` |
| 7 | Trunk はその VLAN を**許可**しているか | `show interfaces <if> trunk` |

---

## 1. VLAN 割り当ての問題

![スイッチ1台のトポロジー](../images/troubleshooting/topology-single-switch.png)

VPC11 (192.168.1.1/24) から VPC12 (192.168.1.2/24) へ ping が通りません。PC 側に問題はなく、スイッチの問題です。

![ping 失敗](../images/troubleshooting/c1-ping-fail.png)

まず、インターフェースに問題がないか確認します。

![show ip int brief](../images/troubleshooting/c1-ip-int-brief.png)

どちらも up / up です。次に VLAN の割り当てを確認します。

![show vlan](../images/troubleshooting/c1-show-vlan.png)

`show vlan` を見ると、**Gi0/0 は VLAN 1、Gi0/1 は VLAN 2** にあります。同じサブネットなのに **VLAN が違う**ため通信できなかったのです。Gi0/1 を VLAN 1 に移します。

![VLAN 1 に変更](../images/troubleshooting/c1-fix-access-vlan.png)

![ping 成功](../images/troubleshooting/c1-ping-ok.png)

> **教訓**: 同じサブネットなのに通信できないときは、まず `show vlan` で**2つのポートの VLAN が同じか**を確認します。

---

## 2. Switchport モードの問題

同じトポロジーで、別の問題です。インターフェースはどちらも up / up です。VLAN の割り当てを見ます。

![show vlan](../images/troubleshooting/c2-show-vlan.png)

**Gi0/1 は VLAN 10** にありますが、**Gi0/0 はどこにも表示されません。** 考えられる原因は2つです。
- インターフェース自体の故障 → up / up なので違う
- インターフェースが access ではなく **trunk** (`show vlan` は access ポートしか表示しない)

![switchport の確認](../images/troubleshooting/c2-switchport-trunk.png)

Gi0/0 は **trunk モード**で、native VLAN は 1 です。PC は 802.1Q タグを付けないので、VPC11 のトラフィックは **native VLAN 1** に入ります。そのため VLAN 10 の VPC12 と通信できません。access モードに変えて VLAN 10 に入れます。

![access モードに変更](../images/troubleshooting/c2-fix-access-mode.png)

![show vlan id 10](../images/troubleshooting/c2-show-vlan-id-10.png)

これで2つのポートとも VLAN 10 にあります。動作モードも確認します。

![Operational Mode の確認](../images/troubleshooting/c2-operational-mode.png)

![ping 成功](../images/troubleshooting/c2-ping-ok.png)

> **教訓**: ポートが**正しい switchport モード(access / trunk)**か確認します。`show vlan` にポートが表示されなければ、trunk の可能性が高いです。

---

## 3. VACL (VLAN Access-List) の問題

同じ PC、同じスイッチで、さらに別の問題です。1つずつ確認していきます。

![show ip int brief](../images/troubleshooting/c3-ip-int-brief.png)

インターフェースは up / up。VLAN の割り当ては?

![show vlan](../images/troubleshooting/c3-show-vlan.png)

2つのポートとも **VLAN 10**。では Port Security は?

![port-security](../images/troubleshooting/c3-port-security.png)

Port Security もありません。インターフェース正常、同じ VLAN、Port Security なし… それでも通らないなら **VACL** を疑います。VACL は**同じ VLAN 内**のトラフィックを許可・遮断できます。

![show vlan filter](../images/troubleshooting/c3-vlan-filter.png)

VLAN 10 に `BLOCKSTUFF` という VLAN access-map が適用されています。

![show vlan access-map](../images/troubleshooting/c3-access-map.png)

![show access-lists](../images/troubleshooting/c3-access-list.png)

- access-map の動作(Action)は **drop** です。
- 一致条件は access-list 1 で、その中の `permit 192.168.1.0 0.0.0.255` は「許可」ではなく **「この範囲に一致する」** という意味です。
- 2台の PC はこの範囲にあるので一致 → **drop** されます。

動作を **forward** に変えます。

![action forward](../images/troubleshooting/c3-fix-action-forward.png)

![ping 成功](../images/troubleshooting/c3-ping-ok.png)

> **教訓**: 他はすべて正常なのに同じ VLAN 同士で通信できないときは **VACL** を確認します。VACL 内の ACL の `permit` は「一致」という意味で、実際の許可・遮断は access-map の **action** が決めます。

---

## 4. Trunk のカプセル化の問題

ここまではスイッチ1台の中の問題でした。トラフィックが複数のスイッチを通るなら、**Trunk** も確認する必要があります。

![スイッチ2台のトポロジー](../images/troubleshooting/topology-two-switches.png)

両方のスイッチに VLAN 10 の PC が1台ずつありますが、通信できません。まず SW1 の PC 側ポートを見ます。

![SW1 int g0/0](../images/troubleshooting/c4-sw1-int-g0-0.png)

![SW1 g0/0 switchport](../images/troubleshooting/c4-sw1-g0-0-switchport.png)

![SW1 port-security](../images/troubleshooting/c4-sw1-port-security.png)

up、access、**VLAN 10**、Port Security なし。SW2 でも同じことを確認します。

![SW2 int g0/0](../images/troubleshooting/c4-sw2-int-g0-0.png)

![SW2 g0/0 switchport](../images/troubleshooting/c4-sw2-g0-0-switchport.png)

![SW2 port-security](../images/troubleshooting/c4-sw2-port-security.png)

PC 側のポートは両方とも正常です。ここでできることは2つです。
- 同じスイッチに VLAN 10 の PC をもう1台つなぎ、スイッチ内では通信できるか確認する
- **SW1 ↔ SW2 間のリンク**を確認する ← 問題が起きやすい場所

![SW1 int g0/1](../images/troubleshooting/c4-sw1-int-g0-1.png)

![SW2 int g0/1](../images/troubleshooting/c4-sw2-int-g0-1.png)

インターフェースは正常です。switchport の情報を見ます。

![SW1 g0/1 ISL](../images/troubleshooting/c4-sw1-g0-1-isl.png)

![SW2 g0/1 dot1q](../images/troubleshooting/c4-sw2-g0-1-dot1q.png)

どちらも trunk ですが、**SW1 は ISL、SW2 は 802.1Q** です。カプセル化が違うと、相手が送ったフレームを解釈できません。SW1 を 802.1Q に合わせます。

![カプセル化の変更](../images/troubleshooting/c4-fix-encapsulation.png)

![ping 成功](../images/troubleshooting/c4-ping-ok.png)

> **教訓**: Trunk の両端は**同じカプセル化**を使う必要があります。
> また、スイッチのモデルによってはデフォルトのモードが `dynamic auto` で、**両端とも `dynamic auto` だと access** になります。Trunk は `switchport mode trunk` で明示するのが安全です。([05. DTP](../05-Dynamic-Trunking-Protocol.md))

---

## 5. Trunk の許可 VLAN の問題

同じトポロジーです。次の項目はすでに確認済みで、問題ないとします。
- インターフェース (speed / duplex)
- Port Security
- Switchport の設定 (VLAN の割り当て、access モード)

![ping 失敗](../images/troubleshooting/c5-ping-fail.png)

スイッチ間の Trunk を見ます。

![SW1 g0/1 switchport](../images/troubleshooting/c5-sw1-g0-1-switchport.png)

![SW2 g0/1 switchport](../images/troubleshooting/c5-sw2-g0-1-switchport.png)

どちらも trunk、どちらも 802.1Q。ここまでは正常です。しかし Trunk が動作しているからといって、**すべての VLAN が通れるとは限りません。**

![SW1 の許可 VLAN](../images/troubleshooting/c5-sw1-trunk-allowed.png)

![SW2 の許可 VLAN](../images/troubleshooting/c5-sw2-trunk-allowed.png)

`Vlans allowed on trunk` が **20** だけです。VLAN 10 はこの Trunk を通れません。すべての VLAN を許可します。

![SW1 allowed vlan all](../images/troubleshooting/c5-sw1-fix-allowed.png)

![SW2 allowed vlan all](../images/troubleshooting/c5-sw2-fix-allowed.png)

![ping 成功](../images/troubleshooting/c5-ping-ok.png)

> **教訓**: Trunk が**必要な VLAN をすべて許可しているか**を必ず確認します。
> 実務では `all` ではなく必要な VLAN だけを許可することが多いです。既存のリストに**追加**するには `switchport trunk allowed vlan add 10` を使います。`add` なしで `allowed vlan 10` と入力するとリストが**上書き**され、他の VLAN が切れることがあります。

---

## 修正コマンドのまとめ

| 状況 | 原因 | 解決コマンド |
| --- | --- | --- |
| 1. VLAN 割り当て | 2つのポートの VLAN が違う | `switchport access vlan 1` |
| 2. Switchport モード | PC のポートが trunk | `switchport mode access` → `switchport access vlan 10` |
| 3. VACL | access-map の action が drop | `vlan access-map BLOCKSTUFF 10` → `action forward` |
| 4. Trunk のカプセル化 | ISL ↔ 802.1Q の不一致 | `switchport trunk encapsulation dot1q` |
| 5. 許可 VLAN | Trunk が VLAN 20 だけを許可 | `switchport trunk allowed vlan all` |

## まとめ

- **下の層から**確認します: インターフェース → VLAN 割り当て → switchport モード → Port Security → VACL → Trunk。
- `show vlan` にポートがなければ **trunk** かもしれません。
- 同じ VLAN なのに通らないなら **VACL** を疑います。
- Trunk は**両端のモードとカプセル化**、そして**許可 VLAN のリスト**をあわせて確認します。
