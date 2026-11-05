# Introduction to VLAN Trunking Protocol (VTP)

[한국어](../06-Introduction-to-VTP.md) | [English](06-Introduction-to-VTP.en.md) | **日本語** | [简体中文](06-Introduction-to-VTP.zh-CN.md) | [Español](06-Introduction-to-VTP.es.md) | [Français](06-Introduction-to-VTP.fr.md)

スイッチが20台、VLAN が50個あるなら、すべてのスイッチに VLAN を1つずつ手作業で作るのは時間がかかりすぎます。**VTP (VLAN Trunking Protocol)** を使えば、1台のスイッチで VLAN を作るだけで、残りのスイッチがその情報を受け取って**自動で同期**します。

- Cisco 独自のプロトコル
- VTP の情報は **Trunk リンク**でのみ伝わります
- 同じ **VTP ドメイン名**を持つスイッチ同士だけが同期します

---

## 1. VTP の仕組み

![VTP domain](../images/vtp-domain.svg)

1. **VTP Server** で VLAN を追加 / 変更 / 削除します。
2. 変更のたびに **Configuration Revision** 番号が1ずつ増えます。
3. 最新の情報(advertisement)がすべての VTP Client に送られます。
4. VTP Client は、**自分より revision が高い**情報を受け取るとその内容に同期します。

> VTP Server も同期します(Client の役割も兼ねる)。そのため VTP Server を複数台置き、どの Server でも VLAN を変更できます。

---

## 2. VTP のモード

### VTP Transparent

![VTP transparent](../images/vtp-transparent.svg)

VTP Transparent は advertisement を他のスイッチへ**転送するだけ**で、**自分は同期しません**。その代わり、VLAN を**自分のスイッチだけで**作成できます。

VTP Server で VLAN 20 を作ると:
1. VTP Server で VLAN 20 を作成
2. revision 番号が増加
3. VTP Server が advertisement を Transparent スイッチへ送信
4. Transparent は同期せず、advertisement を Client へ**転送**
5. Client が最新の情報に同期

### モードの比較

| | **VTP Server** | **VTP Client** | **VTP Transparent** |
| --- | --- | --- | --- |
| VLAN の作成 / 変更 / 削除 | ✅ | ❌ | 自分のスイッチのみ |
| 自分で同期する | ✅ | ✅ | ❌ |
| advertisement の転送 | ✅ | ✅ | ✅ |
| VLAN 情報の保存場所 | `vlan.dat` | `vlan.dat` | **running-config** (+ `vlan.dat`) |

> このほか、VTP を完全に無効にする **`off` モード**もあります。Transparent に似ていますが、advertisement の**転送もしません**(VTP バージョン3、またはそれに対応した IOS)。

---

## 3. VTP の危険: revision 番号

VTP は便利ですが、大きなリスクがあります。**VTP Server も Client のように同期**し、すべてのスイッチは **revision 番号がいちばん高い情報**に従います。

次のようなことが起こり得ます。

1. 本番ネットワークから VTP Client のスイッチを1台外して実習室へ持っていく
2. VTP Server モードに変更する
3. VLAN を作ったり変更したりしてテスト → 変更のたびに **revision が増加**
4. テストが終わったので **VLAN をすべて削除**
5. 再び VTP Client モードに戻す
6. そのスイッチを**本番ネットワークに再接続**

このスイッチの revision は本番ネットワークより**高い**ため、他のスイッチがこの情報に同期し → **本番の VLAN がすべて消えます。** その VLAN に属していたポートは VLAN 1 にも戻らず、存在しない VLAN に残ったまま**通信が止まります。**

> **予防策**
> - 他の場所で使っていたスイッチを接続する前に **revision を 0 にリセット**します。VTP ドメイン名を一時的に別の名前に変えてから戻すか、Transparent モードに変えてから戻します。
> - `vtp password` でドメインにパスワードを設定します。
> - **VTP バージョン3** では、**Primary Server** に指定したスイッチだけが VLAN 情報を変更できるため、この事故を防げます。
> - VTP が不要なら **Transparent**(または `off`)で運用する現場も多くあります。

---

## 4. VTP プルーニング

![VTP pruning](../images/vtp-pruning.svg)

VLAN 10、20、30 に PC があり、スイッチ間はすべての VLAN を運ぶ **802.1Q Trunk** です。
VLAN 10 の PC が**ブロードキャスト**を送ると、スイッチはフラッディングし、Trunk はすべての VLAN を運ぶので、**すべてのスイッチに広がります。**

しかし真ん中のスイッチには VLAN 10 の PC が**1台もありません**(VLAN 20 だけ)。このブロードキャストは**帯域の無駄**です。

**VTP プルーニング**を有効にすると、その VLAN を使う機器がない側の Trunk には、その VLAN のトラフィックを送らなくなります。

```
Switch(config)#vtp pruning
```

- Server で有効にすると、ドメイン全体に適用されます。
- スイッチのモデルによって、デフォルトは On または Off です。
- VLAN 1 と 1002 ~ 1005 はプルーニングされません。

---

## 5. VTP 設定の実習

![VTP lab topology](../images/vtp-lab-topology.svg)

3台のスイッチ(SW1、SW2、SW3)を Trunk で接続し、すべてのスイッチの VLAN データベースと startup-config を消去した状態から始めます。

### 5-1. 初期状態の確認

```
SW1#show vtp status
VTP Version capable             : 1 to 3
VTP version running             : 1
VTP Domain Name                 :
VTP Pruning Mode                : Disabled
VTP Traps Generation            : Disabled
Configuration last modified by 0.0.0.0 at 0-0-00 00:00:00

Feature VLAN:
--------------
VTP Operating Mode                : Server
Maximum VLANs supported locally   : 1005
Number of existing VLANs          : 5
Configuration Revision            : 0
```

SW2、SW3 も同じです。

| 項目 | 意味 |
| --- | --- |
| `Configuration Revision : 0` | VLAN を追加・削除するたびに増える。まだ何も変えていないので 0 |
| `VTP Operating Mode : Server` | デフォルトのモードは **Server** |
| `VTP Pruning Mode : Disabled` | プルーニングは無効 |
| `VTP version running : 1` | バージョン1~3に対応しているが、現在はバージョン1で動作 |
| `VTP Domain Name :` (空欄) | まだドメイン名がない |

### 5-2. SW1 で VLAN を作成

```
SW1(config)#vlan 10
SW1(config-vlan)#name Printers
```

```
SW1#show vlan
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi0/0, Gi0/1, Gi0/2, Gi0/3
                                                Gi1/0, Gi1/1, Gi1/2, Gi1/3
10   Printers                         active
```

```
SW1#show vtp status | include Revision
Configuration Revision            : 1
```

SW1 の revision が **1** に上がりました。しかし SW2、SW3 はまだ **0** で、VLAN 10 もありません。
→ **VTP ドメイン名**が設定されていないためです。

### 5-3. VTP ドメインの設定

リアルタイムで確認するため、SW2、SW3 で debug を有効にします。

```
SW2#debug sw-vlan vtp events
vtp events debugging is on
```

SW1 にドメイン名を設定します。

```
SW1(config)#vtp domain JJONG
Changing VTP domain name from NULL to JJONG
```

SW3 の debug 出力:

```
VTP LOG RUNTIME: Summary packet received in NULL domain state
VTP LOG RUNTIME: Summary packet received, domain = JJONG, rev = 1, followers = 1, length 77, trunk Gi0/0
VTP LOG RUNTIME: Transitioning from NULL to JJONG domain
VTP LOG RUNTIME: Summary packet rev 1 greater than domain JJONG rev 0
```

- ドメイン名が**なかった(NULL)**スイッチは、受け取った VTP パケットのドメイン名(`JJONG`)を**自分のドメインとして採用します。**(すでにドメイン名があれば変えません。)
- 受け取ったパケットの revision(**1**)が自分のもの(**0**)より高いので、**同期します。**

debug は出力が多いので、必ず無効にします。

```
SW2#no debug all
All possible debugging has been turned off
```

```
SW2#show vtp status | include Domain|Revision
VTP Domain Name                 : JJONG
Configuration Revision          : 1
```

```
SW2#show vlan
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi0/1, Gi0/3, Gi1/0, Gi1/1
                                                Gi1/2, Gi1/3
10   Printers                         active
```

SW2、SW3 が VTP で **VLAN 10 を学習しました。**

### 5-4. 複数の Server で VLAN を作成

今は3台とも Server なので、どのスイッチでも VLAN を作れます。

```
SW2(config)#vlan 20
SW2(config-vlan)#name Servers
```
```
SW3(config)#vlan 30
SW3(config-vlan)#name Management
```

```
SW1#show vlan
VLAN Name                             Status
---- -------------------------------- ---------
1    default                          active
10   Printers                         active
20   Servers                          active
30   Management                       active
```

SW2、SW3 も同じく 10、20、30 を持ち、revision は3台とも **3** です。VLAN を作るたびに1ずつ増えます。

### 5-5. VTP Client モード

```
SW2(config)#vtp mode client
Setting device to VTP Client mode for VLANS.
```

```
SW2#show vtp status | include Operating
VTP Operating Mode                : Client
```

これで SW1・SW3 は Server、SW2 は Client です。テストのため **SW1 ↔ SW3 のリンクを切断**し、2台の間に直接の接続がないようにします。(SW1 — SW2 — SW3)

```
SW1(config)#vlan 40
SW1(config-vlan)#name Engineering
```

```
SW2#show vlan
VLAN Name                             Status
---- -------------------------------- ---------
1    default                          active
10   Printers                         active
20   Servers                          active
30   Management                       active
40   Engineering                      active
```

SW3 も VLAN 40 を学習します。SW1 → **SW2(Client)** → SW3 の経路で届きました。
→ **Client も自分で同期し、advertisement を他のスイッチへ転送します。**

Client では VLAN を作成できません。

```
SW2(config)#vlan 50
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

### 5-6. VTP Transparent モード

SW1 ↔ SW3 のリンクは切断したまま、SW2 を Transparent に変更します。

```
SW2(config)#vtp mode transparent
Setting device to VTP Transparent mode for VLANS.
```

SW1 で VLAN 50 を作成します。

```
SW1(config)#vlan 50
SW1(config-vlan)#name Research
```

| スイッチ | VLAN 50 | 理由 |
| --- | --- | --- |
| SW1 (Server) | ✅ あり | 自分で作成 |
| SW2 (Transparent) | ❌ なし | Transparent は**同期しない** |
| SW3 (Server) | ✅ あり | SW2 が advertisement を**転送**した |

今度は SW2(Transparent)で VLAN 60 を作成します。

```
SW2(config)#vlan 60
SW2(config-vlan)#name Cameras
```

```
SW2#show vlan
VLAN Name                             Status
---- -------------------------------- ---------
1    default                          active
10   Printers                         active
20   Servers                          active
30   Management                       active
40   Engineering                      active
60   Cameras                          active
```

Transparent は**自分のスイッチでは** VLAN を作成できます。しかし SW1、SW3 には VLAN 60 が**表示されません。** Transparent の VLAN はそのスイッチだけで使われ、アドバタイズされないためです。

### 5-7. Transparent は VLAN を running-config に保存

```
SW2#show running-config
...
vlan 10
 name Printers
!
vlan 20
 name Servers
!
vlan 30
 name Management
!
vlan 40
 name Engineering
!
vlan 60
 name Cameras
!
```

| モード | VLAN 情報の保存場所 |
| --- | --- |
| Server / Client | `flash:vlan.dat` (running-config には表示されない) |
| **Transparent** | **running-config** にも表示される → `copy run start` で保存 |

---

## 6. コマンドまとめ

| コマンド | 説明 |
| --- | --- |
| `vtp domain <名前>` | VTP ドメイン名を設定 (大文字と小文字を区別) |
| `vtp mode {server \| client \| transparent \| off}` | VTP モードを変更 |
| `vtp version {1 \| 2 \| 3}` | VTP バージョンを変更 |
| `vtp password <パスワード>` | ドメインのパスワードを設定 |
| `vtp pruning` | VTP プルーニングを有効化 |
| `show vtp status` | バージョン、ドメイン、モード、revision を確認 |
| `show vtp password` | 設定したパスワードを確認 |
| `debug sw-vlan vtp events` | VTP イベントをリアルタイムで確認 (`no debug all` で無効化) |

---

## 7. まとめ

- VTP は1台のスイッチで作った VLAN を、**同じドメインのスイッチへ自動で同期**します。**ドメイン名が同じ**でなければ動作せず、**Trunk** でのみ伝わります。
- **Server**: VLAN 作成可 + 同期 + 転送 / **Client**: 同期 + 転送 / **Transparent**: 転送のみ、VLAN は自分のものだけ
- すべてのスイッチは **revision がいちばん高い**情報に従います。他の場所で使っていたスイッチは、接続前に **revision を 0 に**しないと、VLAN 全削除の事故につながります。
- **VTP プルーニング**は、不要な Trunk への VLAN トラフィックを止めて帯域を節約します。
- Transparent は VLAN を **running-config** に、Server/Client は **vlan.dat** に保存します。
