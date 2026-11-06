# VTP Version 3

[한국어](../07-VTP-Version-3.md) | [English](07-VTP-Version-3.en.md) | **日本語** | [简体中文](07-VTP-Version-3.zh-CN.md) | [Español](07-VTP-Version-3.es.md) | [Français](07-VTP-Version-3.fr.md)

VTP バージョン3の目的も VLAN の同期であることは変わりませんが、バージョン1・2の弱点を改善し、多くの機能が追加されました。以前の Catalyst IOS では対応しておらず、最近の IOS から使えるようになりました。

> VTP の基本(Server / Client / Transparent、revision)は、先に [06. Introduction to VTP](06-Introduction-to-VTP.ja.md) を読んでください。

## VTP バージョン3で変わった点

| 機能 | 説明 |
| --- | --- |
| **Primary Server** | Primary Server だけが VLAN を作成 / 変更 / 削除できる → v1・v2 のような **誤って VLAN がすべて消える事故**を防ぐ |
| **拡張 VLAN** | 拡張範囲の VLAN **1006 ~ 4094** も同期 |
| **Private VLAN** | Private VLAN の情報を同期 |
| **RSPAN VLAN** | Remote SPAN VLAN を同期 |
| **MST 対応** | スイッチごとに入力していた **MST 設定**を同期 |
| **認証の強化** | パスワードを **hidden / secret** で保存できる |
| **VTP mode off** | VTP をグローバルまたはインターフェース単位で**完全に無効化** |
| **互換性** | バージョン**2とは互換**、バージョン1とは互換なし |

---

## 実習トポロジー

![VTP version 3 lab topology](../images/vtp3-topology.svg)

SW1、SW2、SW3 間のリンクはすべて Trunk です。

---

## 1. 基本設定

### 1-1. まずドメイン名

```
SW1(config)#vtp version 3
```

VTP バージョン3は、**ドメイン名が空(NULL)のままでは有効になりません。** 先にドメインを設定します。

```
SW1(config)#vtp domain JJONG
SW1(config)#vtp version 3
```

SW2、SW3 も同じように設定します。3台ともデフォルトの **VTP Server** モードで動作します。

### 1-2. Server だけでは VLAN を作れない

```
SW1(config)#vlan 10
```

VTP バージョン3では、Server であっても **Primary Server でなければ** VLAN データベースを変更できないため、このコマンドは拒否されます。

### 1-3. Primary Server の指定

```
SW1#vtp primary vlan
```

- **設定モードではなく特権モード(`#`)**で実行します。
- 確認(confirm)すると SW1 が **VLAN 機能の Primary Server** になり、他のスイッチにも Primary Server が変わったというログが出ます。

```
SW1#show vtp status
```

`Feature VLAN` の項目で `VTP Operating Mode : Primary Server` と Primary ID(MAC アドレス)を確認できます。SW2、SW3 から見ても **SW1 が Primary Server** と表示されます。

### 1-4. ドメイン内のすべてのスイッチを表示

```
SW1#show vtp devices
```

VTP バージョン3で新しく追加されたコマンドです。CDP のように直接つながった機器だけでなく、**同じ VTP ドメインのすべてのスイッチ**を表示します。

### 1-5. VLAN の同期

```
SW1(config)#vlan 100
```

SW2、SW3 で `show vlan brief` を見ると **VLAN 100** ができています。

### 1-6. 拡張範囲の VLAN (1006 ~ 4094)

```
SW1(config)#vlan 2000
```

```
SW2#show vlan brief | include 2000
```

バージョン1・2 では拡張 VLAN を VTP で送れませんでしたが、バージョン3 では **2000番もそのまま同期**されます。

---

## 2. Private VLAN

VTP バージョン3は Private VLAN の情報を同期します。ただし送るのは **VLAN データベースの情報だけ**で、ポートの設定は送りません。

```
SW1(config)#vlan 501
SW1(config-vlan)#private-vlan community
SW1(config-vlan)#vlan 502
SW1(config-vlan)#private-vlan isolated
SW1(config-vlan)#vlan 500
SW1(config-vlan)#private-vlan primary
SW1(config-vlan)#private-vlan association 501,502
```

| VLAN | 種類 |
| --- | --- |
| 500 | Primary VLAN |
| 501 | Community VLAN |
| 502 | Isolated VLAN |

```
SW2#show vlan private-vlan
```

SW1 で作った Private VLAN の構成が、SW2、SW3 にも**まったく同じ**ように表示されます。

---

## 3. Remote SPAN (RSPAN)

RSPAN VLAN も特別な種類の VLAN ですが、VTP で同期されるようになりました。

```
SW1(config)#vlan 600
SW1(config-vlan)#remote-span
```

```
SW3#show vlan remote-span
```

すべてのスイッチで VLAN 600 が **RSPAN VLAN** として表示されます。

---

## 4. MST (Multiple Spanning-Tree)

以前は同じ MST 設定をスイッチごとに手で入力する必要がありました。VTP バージョン3は MST を別の **feature** として同期します。

| Feature | 同期するもの | デフォルトのモード |
| --- | --- | --- |
| **VLAN** | VLAN、拡張 VLAN、Private VLAN、RSPAN | Server |
| **MST** | MST 設定 | Transparent |
| **UNKNOWN** | 今後追加される機能のための枠 | Transparent |

MST にも **Primary Server** が必要です。VLAN の Primary と同じスイッチでも別のスイッチでもかまいません。ここでは **SW2** を MST の Primary にします。

```
SW2(config)#vtp mode server mst
SW2(config)#exit
SW2#vtp primary mst
```

```
SW1(config)#vtp mode client mst
SW3(config)#vtp mode client mst
```

SW2 で MST 設定を作成します。

```
SW2(config)#spanning-tree mst configuration
SW2(config-mst)#name JJONG
SW2(config-mst)#revision 1
SW2(config-mst)#instance 1 vlan 10,20
SW2(config-mst)#instance 2 vlan 30,40
```

すべてのスイッチで MST を有効にします。

```
SW1(config)#spanning-tree mode mst
SW2(config)#spanning-tree mode mst
SW3(config)#spanning-tree mode mst
```

```
SW1#show spanning-tree mst configuration
```

SW1、SW3 も SW2 で作った MST 設定を受け取り、**running-config にも保存**されます。もう MST 設定をコピー&ペーストする必要はありません。

---

## 5. UNKNOWN feature

`show vtp status` には VLAN、MST 以外に **UNKNOWN** feature もあります。将来 VTP バージョン3 が新しい機能を使うときのための枠です。

- 今は **Transparent モードだけ**が使えます。
- `vtp mode server unknown` や `vtp mode client unknown` はエラーになります。

```
SW1(config)#vtp mode transparent unknown
```

---

## 6. 認証 (Authentication)

バージョン3 ではパスワードを **hidden** で保存できます。

```
SW1(config)#vtp password <PASSWORD> hidden
```

```
SW1#show vtp password
```

hidden に設定すると、パスワードは平文ではなく**暗号化された16進数の文字列**としてだけ表示され、スイッチから元のパスワードを取り出せません。
新しいスイッチをドメインに加えるときは、その文字列を **secret** としてそのまま入力します。

```
SW4(config)#vtp password <encrypted-string> secret
```

> Primary Server に指定するとき(`vtp primary`)もパスワードを聞かれます。hidden パスワードは Primary の変更まで防いでくれます。

---

## 7. VTP mode off

バージョン3は VTP を**完全に無効にする** `off` モードに対応しています。

| モード | 自分で同期 | advertisement の転送 |
| --- | --- | --- |
| Transparent | ❌ | ✅ 転送を続ける |
| **Off** | ❌ | ❌ **転送もしない** |

グローバルに無効化 (feature ごとに指定可能):

```
SW1(config)#vtp mode off
SW1(config)#vtp mode off vlan
SW1(config)#vtp mode off mst
```

インターフェース単位で無効化:

```
SW1(config)#interface GigabitEthernet0/1
SW1(config-if)#no vtp
```

このインターフェースは VTP に参加しなくなります。

---

## 8. 下位互換性

- VTP バージョン3は**バージョン2と互換**で、**バージョン1とは互換性がありません。**
- バージョン3 のスイッチがバージョン2 の advertisement を受け取ると、バージョン2 と互換の advertisement を転送します。
- バージョン2 は新しいパスワード方式を理解できないため、混在させるときは**バージョン3 スイッチのパスワードを解除**します。
- 動作がおかしいときは debug で確認します。(元の実習では、古い IOS の Catalyst 3750 で VTP を受信できない問題がありました。)

```
SW1#debug sw-vlan vtp events
SW1#no debug all
```

---

## コマンドまとめ

| コマンド | 説明 |
| --- | --- |
| `vtp domain <名前>` | ドメイン名を設定 (バージョン3では必須) |
| `vtp version 3` | VTP バージョン3に変更 |
| `vtp primary [vlan \| mst]` | Primary Server に指定 (特権モード) |
| `vtp mode {server \| client \| transparent \| off} [vlan \| mst \| unknown]` | feature ごとの VTP モード |
| `vtp password <パスワード> [hidden \| secret]` | パスワードを設定 |
| `no vtp` (interface) | そのインターフェースで VTP を無効化 |
| `show vtp status` | モード、Primary Server、revision を確認 |
| `show vtp devices` | ドメイン内のすべての VTP スイッチを表示 |
| `show vtp password` | パスワードを確認 (hidden なら暗号化文字列) |

---

## まとめ

- VTP バージョン3は**ドメイン名が必須**で、**Primary Server だけ**が VLAN を変更できるため、VLAN 全削除の事故を防ぎます。
- **拡張 VLAN、Private VLAN、RSPAN VLAN、MST 設定**まで同期します。
- 機能が **VLAN / MST / UNKNOWN feature** に分かれ、feature ごとにモードと Primary Server を決めます。
- **hidden パスワード**と **off モード**(グローバル・インターフェース)が追加されました。
- **バージョン2とだけ**互換性があります。
