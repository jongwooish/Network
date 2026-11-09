# How to Troubleshoot VLANs and Trunks

[한국어](../08-How-to-Troubleshoot-VLANs-and-Trunks.md) | [English](08-How-to-Troubleshoot-VLANs-and-Trunks.en.md) | [日本語](08-How-to-Troubleshoot-VLANs-and-Trunks.ja.md) | **简体中文** | [Español](08-How-to-Troubleshoot-VLANs-and-Trunks.es.md) | [Français](08-How-to-Troubleshoot-VLANs-and-Trunks.fr.md)

如果已经检查过接口设置(speed / duplex、错误等)仍然无法通信，问题可能出在 **VLAN 或 Trunk** 上。下面结合实验截图整理 5 种常见情况。

## 排查顺序一览

| 顺序 | 检查内容 | 命令 |
| --- | --- | --- |
| 1 | 接口是否 up / up | `show ip interface brief`、`show interfaces <if>` |
| 2 | 两个端口是否在**同一个 VLAN** | `show vlan` |
| 3 | 端口是否处于**正确的 access / trunk 模式** | `show interfaces <if> switchport` |
| 4 | Port Security 是否在拦截 | `show port-security` |
| 5 | 是否有 **VACL** 拦截 VLAN 内的流量 | `show vlan filter`、`show vlan access-map` |
| 6 | Trunk 两端的**封装是否相同** | `show interfaces <if> switchport` |
| 7 | Trunk 是否**允许**该 VLAN | `show interfaces <if> trunk` |

---

## 1. VLAN 分配问题

![单台交换机拓扑](../images/troubleshooting/topology-single-switch.png)

VPC11 (192.168.1.1/24) 无法 ping 通 VPC12 (192.168.1.2/24)。PC 本身没有问题，问题在交换机上。

![ping 失败](../images/troubleshooting/c1-ping-fail.png)

先检查接口是否有问题。

![show ip int brief](../images/troubleshooting/c1-ip-int-brief.png)

两个接口都是 up / up。接下来检查 VLAN 分配。

![show vlan](../images/troubleshooting/c1-show-vlan.png)

`show vlan` 显示 **Gi0/0 在 VLAN 1，Gi0/1 在 VLAN 2**。虽然在同一子网，但 **VLAN 不同**，所以无法通信。把 Gi0/1 移到 VLAN 1。

![改为 VLAN 1](../images/troubleshooting/c1-fix-access-vlan.png)

![ping 成功](../images/troubleshooting/c1-ping-ok.png)

> **经验**: 同一子网却无法通信时，先用 `show vlan` 确认**两个端口的 VLAN 是否相同**。

---

## 2. Switchport 模式问题

同样的拓扑，不同的问题。两个接口都是 up / up。查看 VLAN 分配。

![show vlan](../images/troubleshooting/c2-show-vlan.png)

**Gi0/1 在 VLAN 10**，但 **Gi0/0 哪里都看不到。** 可能的原因有两个:
- 接口本身故障 → 是 up / up，所以不是
- 接口不是 access 而是 **trunk** (`show vlan` 只显示 access 端口)

![查看 switchport](../images/troubleshooting/c2-switchport-trunk.png)

Gi0/0 处于 **trunk 模式**，native VLAN 为 1。PC 不会打 802.1Q 标签，所以 VPC11 的流量进入 **native VLAN 1**，无法与 VLAN 10 的 VPC12 通信。改为 access 模式并放入 VLAN 10。

![改为 access 模式](../images/troubleshooting/c2-fix-access-mode.png)

![show vlan id 10](../images/troubleshooting/c2-show-vlan-id-10.png)

现在两个端口都在 VLAN 10。再确认一下工作模式。

![查看 Operational Mode](../images/troubleshooting/c2-operational-mode.png)

![ping 成功](../images/troubleshooting/c2-ping-ok.png)

> **经验**: 确认端口处于**正确的 switchport 模式(access / trunk)**。如果 `show vlan` 中看不到某个端口，它很可能是 trunk。

---

## 3. VACL (VLAN Access-List) 问题

同样的 PC、同样的交换机，又一个问题。逐项检查。

![show ip int brief](../images/troubleshooting/c3-ip-int-brief.png)

接口 up / up。VLAN 分配呢?

![show vlan](../images/troubleshooting/c3-show-vlan.png)

两个端口都在 **VLAN 10**。那 Port Security 呢?

![port-security](../images/troubleshooting/c3-port-security.png)

也没有 Port Security。接口正常、同一 VLAN、没有 Port Security… 如果仍然不通，就要怀疑 **VACL**。VACL 可以允许或阻止**同一 VLAN 内**的流量。

![show vlan filter](../images/troubleshooting/c3-vlan-filter.png)

VLAN 10 上应用了名为 `BLOCKSTUFF` 的 VLAN access-map。

![show vlan access-map](../images/troubleshooting/c3-access-map.png)

![show access-lists](../images/troubleshooting/c3-access-list.png)

- access-map 的动作(Action)是 **drop**。
- 匹配条件是 access-list 1，其中的 `permit 192.168.1.0 0.0.0.255` 不是“允许”，而是 **“匹配这个范围”** 的意思。
- 两台 PC 都在这个范围内，所以被匹配 → **drop**。

把动作改为 **forward**。

![action forward](../images/troubleshooting/c3-fix-action-forward.png)

![ping 成功](../images/troubleshooting/c3-ping-ok.png)

> **经验**: 其他都正常、同一 VLAN 内却无法通信时，要检查 **VACL**。VACL 中 ACL 的 `permit` 表示“匹配”，真正决定转发还是丢弃的是 access-map 的 **action**。

---

## 4. Trunk 封装问题

前面都是单台交换机内部的问题。如果流量要经过多台交换机，还需要检查 **Trunk**。

![两台交换机拓扑](../images/troubleshooting/topology-two-switches.png)

两台交换机上各有一台 VLAN 10 的 PC，但互相无法通信。先查看 SW1 连接 PC 的端口。

![SW1 int g0/0](../images/troubleshooting/c4-sw1-int-g0-0.png)

![SW1 g0/0 switchport](../images/troubleshooting/c4-sw1-g0-0-switchport.png)

![SW1 port-security](../images/troubleshooting/c4-sw1-port-security.png)

up、access、**VLAN 10**、没有 Port Security。在 SW2 上检查同样的内容。

![SW2 int g0/0](../images/troubleshooting/c4-sw2-int-g0-0.png)

![SW2 g0/0 switchport](../images/troubleshooting/c4-sw2-g0-0-switchport.png)

![SW2 port-security](../images/troubleshooting/c4-sw2-port-security.png)

两边连接 PC 的端口都正常。现在可以做两件事:
- 在同一台交换机上再接一台 VLAN 10 的 PC，确认交换机内部能否通信
- 检查 **SW1 ↔ SW2 之间的链路** ← 最容易出问题的地方

![SW1 int g0/1](../images/troubleshooting/c4-sw1-int-g0-1.png)

![SW2 int g0/1](../images/troubleshooting/c4-sw2-int-g0-1.png)

接口正常。查看 switchport 信息。

![SW1 g0/1 ISL](../images/troubleshooting/c4-sw1-g0-1-isl.png)

![SW2 g0/1 dot1q](../images/troubleshooting/c4-sw2-g0-1-dot1q.png)

两边都是 trunk，但 **SW1 用 ISL，SW2 用 802.1Q**。封装不同，就无法解析对方发来的帧。把 SW1 改为 802.1Q。

![修改封装](../images/troubleshooting/c4-fix-encapsulation.png)

![ping 成功](../images/troubleshooting/c4-ping-ok.png)

> **经验**: Trunk 两端必须使用**相同的封装**。
> 另外，根据交换机型号，默认模式可能是 `dynamic auto`，而**两端都是 `dynamic auto` 时会变成 access**。最安全的做法是用 `switchport mode trunk` 明确指定 Trunk。([05. DTP](../05-Dynamic-Trunking-Protocol.md))

---

## 5. Trunk 允许 VLAN 问题

同样的拓扑。假设以下项目已经检查过且没有问题:
- 接口 (speed / duplex)
- Port Security
- Switchport 配置 (VLAN 分配、access 模式)

![ping 失败](../images/troubleshooting/c5-ping-fail.png)

查看交换机之间的 Trunk。

![SW1 g0/1 switchport](../images/troubleshooting/c5-sw1-g0-1-switchport.png)

![SW2 g0/1 switchport](../images/troubleshooting/c5-sw2-g0-1-switchport.png)

两边都是 trunk，都用 802.1Q，到这里都正常。但 Trunk 正常工作**并不代表所有 VLAN 都能通过。**

![SW1 允许的 VLAN](../images/troubleshooting/c5-sw1-trunk-allowed.png)

![SW2 允许的 VLAN](../images/troubleshooting/c5-sw2-trunk-allowed.png)

`Vlans allowed on trunk` 只有 **20**。VLAN 10 无法通过这条 Trunk。允许所有 VLAN。

![SW1 allowed vlan all](../images/troubleshooting/c5-sw1-fix-allowed.png)

![SW2 allowed vlan all](../images/troubleshooting/c5-sw2-fix-allowed.png)

![ping 成功](../images/troubleshooting/c5-ping-ok.png)

> **经验**: 一定要确认 Trunk **是否允许所需的全部 VLAN**。
> 实际工作中常常只允许需要的 VLAN 而不是 `all`。如果要在现有列表中**添加**，请使用 `switchport trunk allowed vlan add 10`。不加 `add` 直接输入 `allowed vlan 10` 会**覆盖**原有列表，可能导致其他 VLAN 中断。

---

## 修复命令汇总

| 情况 | 原因 | 解决命令 |
| --- | --- | --- |
| 1. VLAN 分配 | 两个端口的 VLAN 不同 | `switchport access vlan 1` |
| 2. Switchport 模式 | PC 端口是 trunk | `switchport mode access` → `switchport access vlan 10` |
| 3. VACL | access-map 的 action 是 drop | `vlan access-map BLOCKSTUFF 10` → `action forward` |
| 4. Trunk 封装 | ISL ↔ 802.1Q 不一致 | `switchport trunk encapsulation dot1q` |
| 5. 允许的 VLAN | Trunk 只允许 VLAN 20 | `switchport trunk allowed vlan all` |

## 要点总结

- **从底层开始**检查: 接口 → VLAN 分配 → switchport 模式 → Port Security → VACL → Trunk。
- `show vlan` 中看不到端口时，它可能是 **trunk**。
- 同一 VLAN 仍然不通时，要怀疑 **VACL**。
- Trunk 要同时检查**两端的模式与封装**以及**允许的 VLAN 列表**。
