# Rapid Spanning-Tree (RSTP)

[한국어](../01-Rapid-Spanning-Tree.md) | [English](01-Rapid-Spanning-Tree.en.md) | [日本語](01-Rapid-Spanning-Tree.ja.md) | **简体中文** | [Español](01-Rapid-Spanning-Tree.es.md) | [Français](01-Rapid-Spanning-Tree.fr.md)

如今网络中的路由越来越多。OSPF、EIGRP 等路由协议对网络变化的响应比生成树快得多。为了跟上它们的速度,就有了 **快速生成树(RSTP,IEEE 802.1w)**。

RSTP 是对原有生成树的**演进,而不是革命**。只是为了加快速度改动了一些内部机制,**配置方法保持不变**。下文把原有的生成树(802.1D)称为 **Classic Spanning Tree(经典生成树)**。

## 差异一览

| 项目 | Classic STP (802.1D) | RSTP (802.1w) |
| --- | --- | --- |
| 端口状态 | Blocking / Listening / Learning / Forwarding | **Discarding** / Learning / Forwarding |
| 端口角色 | Root、Designated、Alternate(阻塞) | Root、Designated、Alternate、**Backup** |
| BPDU | Flags 只用 2 位(TC、TCA) | **Version 2 BPDU**,Flags 全部使用 |
| BPDU 由谁发送 | 只有根桥生成,其他交换机转发 | **所有交换机每个 hello time(2 秒)**都生成 |
| 故障检测 | Max Age 20 秒 | **连续丢失 3 个 BPDU** 即检测(keepalive) |
| Inferior BPDU | 忽略 | **接受**(类似 BackboneFast) |
| 进入转发 | 基于计时器(Listening → Learning,约 30 秒) | **Proposal / Agreement 协商(Sync)** |
| PortFast | PortFast | **Edge port** |
| UplinkFast | 需手动配置 | **默认内置** |
| 拓扑变化 | TCN 发送到根桥,再由根桥通知 | **检测到的交换机立即泛洪** |

---

## 1. 端口状态:4 个 → 3 个

![Port states](../images/rstp-port-states.svg)

RSTP 只有三种端口状态:**Discarding、Learning、Forwarding**。**Discarding** 合并了经典的 Blocking 和 Listening 状态。

| Classic Spanning Tree | Rapid Spanning Tree | 端口在拓扑中活动? | 学习 MAC 地址? |
| --- | --- | --- | --- |
| Blocking | Discarding | No | No |
| Listening | Discarding | Yes | No |
| Learning | Learning | Yes | Yes |
| Forwarding | Forwarding | Yes | Yes |

---

## 2. 端口角色

![Port roles](../images/rstp-port-roles.svg)

- **根桥**:Bridge ID(Priority + MAC)最优的交换机,这里是 SW1(MAC AAA)。
- **Root port**:非根交换机上到达根桥开销最低的端口。
- **Designated port**:每个网段只有一个,**能发送最优 BPDU** 的端口成为指定端口。
- **Alternate port**:SW3 的 Fa0/16 从 SW2 收到更优的 BPDU,因此被阻塞。它是根端口的备用路径。

到这里都和经典生成树相同。RSTP 新增的角色是 **Backup port**。

### Backup port

![Backup port](../images/rstp-backup-port.svg)

在 SW2 和 SW3 之间加了一台**集线器(Hub)**。如果没有集线器,SW2 的 Fa0/16、Fa0/17 都会是指定端口。

但因为集线器,SW2 的 Fa0/16 和 Fa0/17 处于**同一个冲突域**。SW2 在 Fa0/16 和 Fa0/17 上都收到了自己发出的 BPDU,因此知道自己有两条连接到同一网段。

- Fa0/16 → **Designated port**
- Fa0/17 → Fa0/16 的 **Backup port**

> 在生产网络中几乎看不到 Backup port。去掉集线器后,两个端口又都会成为指定端口。

---

## 3. BPDU(Version 2)

![RSTP BPDU](../images/rstp-bpdu-flags.svg)

经典生成树只使用 Flags 字段中的**两位**:
- Topology Change
- Topology Change Acknowledgement

RSTP 使用 **Flags 字段的全部位**。特别是通过 **Port Role** 字段告知发送该 BPDU 的端口角色:
- Unknown
- Alternate / Backup port
- Root port
- Designated port

这种新 BPDU 称为 **Version 2 BPDU**。只运行经典生成树的交换机会丢弃它。

---

## 4. 其他变化

### 所有交换机每个 hello time 都发送 BPDU
在经典生成树中,**只有根桥**生成 BPDU,非根交换机只转发从根端口收到的 BPDU。在 RSTP 中,**所有交换机每 2 秒(hello time)**都自己生成 BPDU。hello time 可以修改。

### BPDU = keepalive
经典生成树在丢弃 BPDU 前要等待 **Max Age(20 秒)**。RSTP 像 OSPF、EIGRP 的 hello 一样把 BPDU 当作 **keepalive**。如果**连续丢失邻居交换机的 3 个 BPDU**,就认为连接已断开,并立即删除所有 MAC 地址。

### 接受 Inferior BPDU
经典生成树忽略 inferior BPDU,RSTP 则会接受。这几乎就是经典生成树的 **BackboneFast** 功能。

### 用协商代替计时器
经典生成树在转发前必须经过 Listening、Learning,默认计时器下需要 **30 秒**。RSTP 不用计时器,而是通过**协商**决定端口能否转发。这是 RSTP 最重要的变化:**快速收敛**。

### PortFast = Edge port
在经典生成树中启用 PortFast 后,端口会跳过 Listening、Learning 直接进入 Forwarding,端口 up/down 时也不会产生拓扑变化。RSTP 仍然使用 PortFast,但称为 **Edge port**。

### 链路类型
RSTP 只能在 **Edge port** 和 **Point-to-point** 链路上快速进入转发状态。

| 链路类型 | 双工 | 说明 |
| --- | --- | --- |
| Point-to-point | 全双工 | 交换机之间的普通链路 |
| Shared | 半双工 | 中间接有集线器的链路 |

---

## 5. Sync:Proposal / Agreement

![RSTP sync overview](../images/rstp-sync-overview.svg)

SW1 是根桥,SW2、SW3、SW4 是非根桥。

1. SW1–SW2 链路一建立,两端端口都处于**阻塞状态**。SW2 收到 SW1 的 BPDU 后,就开始一种叫 **Sync** 的协商。
2. SW2 一收到根桥的 BPDU,就**阻塞所有非边缘(non-edge)的指定端口**(non-edge = 通往其他交换机的端口,edge = 配置了 PortFast 的端口)。随后 SW1–SW2 链路进入 **Forwarding**。
3. SW2 接着与 SW3、SW4 进行 Sync,让这些链路也快速进入转发状态。

按顺序看具体交换了哪些消息:

![Proposal and agreement](../images/rstp-proposal-agreement.svg)

| 步骤 | 动作 |
| --- | --- |
| 1. Proposal | SW1 在 BPDU Flags 中置位 **Proposal 位**后发送,SW2 得知 SW1 是根桥 |
| 2. Sync | SW2 **阻塞所有 non-edge 端口**,准备向 SW3、SW4 同步 |
| 3. Agreement | SW2 发送 **Agreement**(Proposal BPDU 的副本,Proposal OFF、Agreement ON)。SW2 Fa0/14 进入转发 |
| 4. Forwarding | SW1 收到 Agreement 后,**立即**将 Fa0/14 置为转发 |
| 5. Proposal | SW2 向 Fa0/16(SW3)、Fa0/19(SW4)发送同样的 Proposal |
| 6. Agreement | SW3、SW4 没有其他端口,立即回复 Agreement |
| 7. Forwarding | SW2 把 Fa0/16、Fa0/19 置为转发,完成 |

> 只是几个消息来回,所以比经典的计时器方式(Listening → Learning → Forwarding)**快得多**。

---

## 6. UplinkFast 默认内置

经典生成树需要手动启用 **UplinkFast**,RSTP 默认就有。交换机失去根端口时,会**立即把 Alternate port 置为转发**。

经典的 UplinkFast 需要发送组播帧来更新所有交换机的 MAC 地址表。RSTP 的拓扑变化机制不同,所以不再需要。

---

## 7. 拓扑变化

![Topology change](../images/rstp-topology-change.svg)

在经典生成树中,**链路故障也会触发拓扑变化**。RSTP 不把链路故障视为拓扑变化,**只有 non-edge 端口(通往其他交换机)进入转发状态时**才算拓扑变化。

**检测到**拓扑变化的交换机会:
- 对所有 non-edge 指定端口和根端口启动 **topology change while 计时器**,值为 **hello time 的 2 倍**。
- **清除**在这些端口上学到的 **MAC 地址**。
- 计时器运行期间,在从这些端口发出的 BPDU 中置位 **TC 位**,也会从根端口发送 BPDU。

**收到** TC 位置位 BPDU 的邻居交换机会:
- 清除**除收到该 BPDU 的端口以外所有端口**上的 MAC 地址。
- 自己也启动 topology change while 计时器,并向所有指定端口和根端口发送 TC 位置位的 BPDU。

> 不像经典生成树那样先上报到根桥再下发,而是**快速泛洪到整个网络**。

---

## 8. 与经典生成树的兼容性

![Compatibility](../images/rstp-compatibility.svg)

RSTP 与经典生成树**是兼容的**。但当 RSTP 交换机与经典交换机相连时,**让 RSTP 变快的功能都无法工作**。

图中 SW1–SW2 之间运行 RSTP,而 SW2–SW3 之间**回退为经典生成树**。

---

## 要点总结

- RSTP(802.1w)是经典 STP 的**演进**,配置相同,内部更快。
- 端口状态为 **Discarding / Learning / Forwarding** 三种,新增角色 **Backup port**。
- **Version 2 BPDU**:使用全部 Flags,包含 Port Role。所有交换机每 2 秒发送一次,连续丢失 3 个即视为故障。
- 用 **Proposal → Sync → Agreement** 协商代替计时器来收敛。快速转换只发生在 **Edge port** 和 **Point-to-point** 链路上。
- **UplinkFast 默认内置**。拓扑变化**只在 non-edge 端口进入转发时**发生,并通过**泛洪**快速传播。
- 与经典 STP 兼容,但相连的区段按经典方式运行。
