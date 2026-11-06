# VTP Version 3

[한국어](../07-VTP-Version-3.md) | [English](07-VTP-Version-3.en.md) | [日本語](07-VTP-Version-3.ja.md) | **简体中文** | [Español](07-VTP-Version-3.es.md) | [Français](07-VTP-Version-3.fr.md)

VTP 版本 3 的目标仍然是同步 VLAN，但它修正了版本 1、2 的缺点，并增加了很多功能。早期的 Catalyst IOS 不支持它，从较新的 IOS 开始才能使用。

> VTP 的基础知识(Server / Client / Transparent、revision)请先阅读 [06. Introduction to VTP](06-Introduction-to-VTP.zh-CN.md)。

## VTP 版本 3 的变化

| 功能 | 说明 |
| --- | --- |
| **Primary Server** | 只有 Primary Server 能创建 / 修改 / 删除 VLAN → 防止 v1、v2 中**误删全部 VLAN** 的事故 |
| **扩展 VLAN** | 扩展范围 VLAN **1006 ~ 4094** 也会同步 |
| **Private VLAN** | 同步 Private VLAN 信息 |
| **RSPAN VLAN** | 同步 Remote SPAN VLAN |
| **支持 MST** | 同步原本需要在每台交换机上手动配置的 **MST 配置** |
| **增强认证** | 密码可以以 **hidden / secret** 方式保存 |
| **VTP mode off** | 可以在全局或接口级别**彻底关闭** VTP |
| **兼容性** | **与版本 2 兼容**，与版本 1 不兼容 |

---

## 实验拓扑

![VTP version 3 lab topology](../images/vtp3-topology.svg)

SW1、SW2、SW3 之间的链路都是 Trunk。

---

## 1. 基本配置

### 1-1. 先配置域名

```
SW1(config)#vtp version 3
```

VTP 版本 3 在**域名为空(NULL)时无法启用。** 先配置域名。

```
SW1(config)#vtp domain JJONG
SW1(config)#vtp version 3
```

SW2、SW3 也做同样的配置。三台交换机都以默认的 **VTP Server** 模式运行。

### 1-2. 只是 Server 还不能创建 VLAN

```
SW1(config)#vlan 10
```

在 VTP 版本 3 中，即使是 Server，**如果不是 Primary Server** 也不能修改 VLAN 数据库，所以这条命令会被拒绝。

### 1-3. 指定 Primary Server

```
SW1#vtp primary vlan
```

- 要在 **特权模式(`#`)而不是配置模式**下执行。
- 确认(confirm)后，SW1 成为 **VLAN 功能的 Primary Server**，其他交换机上也会出现 Primary Server 已变更的日志。

```
SW1#show vtp status
```

在 `Feature VLAN` 项中可以看到 `VTP Operating Mode : Primary Server` 以及 Primary ID(MAC 地址)。在 SW2、SW3 上查看也会显示 **SW1 是 Primary Server**。

### 1-4. 查看域内所有交换机

```
SW1#show vtp devices
```

这是 VTP 版本 3 新增的命令。它不像 CDP 那样只显示直连设备，而是显示**同一 VTP 域内的所有交换机**。

### 1-5. VLAN 同步

```
SW1(config)#vlan 100
```

在 SW2、SW3 上用 `show vlan brief` 查看，可以看到 **VLAN 100** 已经出现。

### 1-6. 扩展范围 VLAN (1006 ~ 4094)

```
SW1(config)#vlan 2000
```

```
SW2#show vlan brief | include 2000
```

版本 1、2 无法通过 VTP 传递扩展 VLAN，但在版本 3 中 **VLAN 2000 也会同步**。

---

## 2. Private VLAN

VTP 版本 3 会同步 Private VLAN 信息。但它**只发送 VLAN 数据库信息**，不发送端口配置。

```
SW1(config)#vlan 501
SW1(config-vlan)#private-vlan community
SW1(config-vlan)#vlan 502
SW1(config-vlan)#private-vlan isolated
SW1(config-vlan)#vlan 500
SW1(config-vlan)#private-vlan primary
SW1(config-vlan)#private-vlan association 501,502
```

| VLAN | 类型 |
| --- | --- |
| 500 | Primary VLAN |
| 501 | Community VLAN |
| 502 | Isolated VLAN |

```
SW2#show vlan private-vlan
```

在 SW1 上创建的 Private VLAN 配置会**完全一样**地出现在 SW2、SW3 上。

---

## 3. Remote SPAN (RSPAN)

RSPAN VLAN 也是一种特殊的 VLAN，现在也可以通过 VTP 同步。

```
SW1(config)#vlan 600
SW1(config-vlan)#remote-span
```

```
SW3#show vlan remote-span
```

所有交换机上的 VLAN 600 都显示为 **RSPAN VLAN**。

---

## 4. MST (Multiple Spanning-Tree)

过去需要在每台交换机上手动输入相同的 MST 配置。VTP 版本 3 把 MST 作为单独的 **feature** 来同步。

| Feature | 同步内容 | 默认模式 |
| --- | --- | --- |
| **VLAN** | VLAN、扩展 VLAN、Private VLAN、RSPAN | Server |
| **MST** | MST 配置 | Transparent |
| **UNKNOWN** | 为未来功能预留 | Transparent |

MST 也需要 **Primary Server**。可以和 VLAN 的 Primary 是同一台交换机，也可以是不同的交换机。这里把 **SW2** 设为 MST 的 Primary。

```
SW2(config)#vtp mode server mst
SW2(config)#exit
SW2#vtp primary mst
```

```
SW1(config)#vtp mode client mst
SW3(config)#vtp mode client mst
```

在 SW2 上创建 MST 配置。

```
SW2(config)#spanning-tree mst configuration
SW2(config-mst)#name JJONG
SW2(config-mst)#revision 1
SW2(config-mst)#instance 1 vlan 10,20
SW2(config-mst)#instance 2 vlan 30,40
```

在所有交换机上启用 MST。

```
SW1(config)#spanning-tree mode mst
SW2(config)#spanning-tree mode mst
SW3(config)#spanning-tree mode mst
```

```
SW1#show spanning-tree mst configuration
```

SW1、SW3 也收到了在 SW2 上创建的 MST 配置，并且**保存到了 running-config 中**。再也不需要复制粘贴 MST 配置了。

---

## 5. UNKNOWN feature

`show vtp status` 中除了 VLAN、MST 之外还有 **UNKNOWN** feature。它是为 VTP 版本 3 将来使用新功能而预留的位置。

- 目前**只能使用 Transparent 模式**。
- `vtp mode server unknown` 或 `vtp mode client unknown` 会报错。

```
SW1(config)#vtp mode transparent unknown
```

---

## 6. 认证 (Authentication)

在版本 3 中，密码可以以 **hidden** 方式保存。

```
SW1(config)#vtp password <PASSWORD> hidden
```

```
SW1#show vtp password
```

设置为 hidden 后，密码只会显示为**加密的十六进制字符串**而不是明文，也无法从交换机中取出原始密码。
新交换机加入域时，把这个字符串作为 **secret** 原样输入。

```
SW4(config)#vtp password <encrypted-string> secret
```

> 指定 Primary Server(`vtp primary`)时也会要求输入密码。hidden 密码连 Primary 的变更也能保护。

---

## 7. VTP mode off

版本 3 支持**彻底关闭** VTP 的 `off` 模式。

| 模式 | 自身同步 | 转发 advertisement |
| --- | --- | --- |
| Transparent | ❌ | ✅ 继续转发 |
| **Off** | ❌ | ❌ **也不转发** |

全局关闭 (可按 feature 指定):

```
SW1(config)#vtp mode off
SW1(config)#vtp mode off vlan
SW1(config)#vtp mode off mst
```

按接口关闭:

```
SW1(config)#interface GigabitEthernet0/1
SW1(config-if)#no vtp
```

该接口不再参与 VTP。

---

## 8. 向下兼容

- VTP 版本 3 **与版本 2 兼容**，**与版本 1 不兼容。**
- 版本 3 交换机收到版本 2 的 advertisement 时，会转发与版本 2 兼容的 advertisement。
- 版本 2 不理解新的密码机制，所以混用时要**取消版本 3 交换机上的密码**。
- 行为异常时用 debug 检查。(在原实验中，旧版 IOS 的 Catalyst 3750 出现了无法接收 VTP 的问题。)

```
SW1#debug sw-vlan vtp events
SW1#no debug all
```

---

## 命令汇总

| 命令 | 说明 |
| --- | --- |
| `vtp domain <名称>` | 设置域名 (版本 3 必需) |
| `vtp version 3` | 切换到 VTP 版本 3 |
| `vtp primary [vlan \| mst]` | 指定 Primary Server (特权模式) |
| `vtp mode {server \| client \| transparent \| off} [vlan \| mst \| unknown]` | 按 feature 设置 VTP 模式 |
| `vtp password <密码> [hidden \| secret]` | 设置密码 |
| `no vtp` (interface) | 在该接口上关闭 VTP |
| `show vtp status` | 查看模式、Primary Server、revision |
| `show vtp devices` | 查看域内所有 VTP 交换机 |
| `show vtp password` | 查看密码 (hidden 时显示加密字符串) |

---

## 要点总结

- VTP 版本 3 **必须配置域名**，并且**只有 Primary Server** 能修改 VLAN，从而防止全部 VLAN 被删除的事故。
- 还会同步**扩展 VLAN、Private VLAN、RSPAN VLAN 和 MST 配置**。
- 功能分为 **VLAN / MST / UNKNOWN feature**，每个 feature 单独设置模式和 Primary Server。
- 新增了 **hidden 密码**和 **off 模式**(全局、接口)。
- **只与版本 2** 兼容。
