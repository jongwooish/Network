# Spanning-Tree BPDU Guard

[한국어](../03-BPDU-Guard.md) | [English](03-BPDU-Guard.en.md) | [日本語](03-BPDU-Guard.ja.md) | **简体中文** | [Español](03-BPDU-Guard.es.md) | [Français](03-BPDU-Guard.fr.md)

**BPDU Guard** 是保护 spanning-tree 拓扑的功能之一。当启用了 BPDU Guard 的接口**收到 BPDU 时，该接口会进入 err-disable 状态**。下面用一个例子说明为什么需要它。

## BPDU Guard 的配置方法

| 位置 | 命令 | 作用范围 |
| --- | --- | --- |
| 接口 | `spanning-tree bpduguard enable` | 该接口 |
| 全局 | `spanning-tree portfast bpduguard default` | 所有启用了 PortFast 的接口 |

---

## 1. 为什么需要 BPDU Guard？

![An attacker sends superior BPDUs](../images/bpduguard-attack.svg)

上面的拓扑中，spanning-tree 工作正常。默认情况下，spanning-tree **在所有接口上收发 BPDU**。在这个例子中，SW2 的 fa0/2 上连着一台计算机。好奇心强或居心不良的人可能会运行一个工具，生成**带有更优（superior）bridge ID 的 BPDU**。于是交换机们会认为 root bridge 现在可以经由 SW2 到达，spanning-tree 就会**重新计算**。这听起来不是好事，对吧？来看看可能出什么问题：

![Man-in-the-middle: the attacker becomes the root](../images/bpduguard-mitm.svg)

甚至可以在没人察觉的情况下发动**中间人攻击（man-in-the-middle）**。设想攻击者把计算机同时连到两台交换机上并成为 root bridge，那么交换机之间的所有流量都会经过攻击者的计算机。只要运行 Wireshark 等着就行了。

用 BPDU Guard 可以防止这种情况，因为一收到 BPDU 它就会阻断接口。

![BPDU Guard drops the attacker](../images/bpduguard-block.svg)

BPDU Guard 在接口上收到 BPDU 时，会把该接口置为 **err-disable** 模式。来看看如何配置。

---

## 2. 在接口上配置

使用下面的拓扑。

![Lab topology: SW2, SW3](../images/bpdu-guard/01-topology.png)

用两台交换机演示 BPDU Guard。配置 SW2 的 Gi0/0，使其在收到 SW3 的 BPDU 时进入 err-disable 模式。

![spanning-tree bpduguard enable on Gi0/0](../images/bpdu-guard/02-bpduguard.png)

这就是在接口上启用的方法。实际中不会在交换机之间的链路上这样做，而应该在**连接计算机的 access 模式接口**上配置。

![BLOCK_BPDUGUARD and err-disable](../images/bpdu-guard/03-err-disable.png)

不出所料，接口被关掉了。Gi0/0 收到 BPDU 后出现 `BLOCK_BPDUGUARD` 消息，进入 **err-disable** 状态，line protocol 也变为 down。删除 BPDU Guard，再对接口执行 `shutdown` / `no shutdown` 使其恢复。

![no spanning-tree bpduguard, shutdown, no shutdown](../images/bpdu-guard/04-remove.png)

---

## 3. 全局配置

除了逐个接口启用，也可以**全局**启用 BPDU Guard。实验设备（IOSv）上没有这个命令，所以这一部分是在 **Packet Tracer** 中完成的。

```
SW2(config)#spanning-tree portfast bpduguard default
```

这条命令会在**所有启用了 PortFast 的接口**上启用 BPDU Guard。因此通常也会一起全局启用 PortFast：

```
SW2(config)#spanning-tree portfast default
```

下面是一个便于验证配置的命令：

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

可以看到 **Portfast Default** 和 **PortFast BPDU Guard Default** 已全局启用。

---

## 总结

- 默认情况下所有接口都会收发 BPDU，所以从主机端口发出 **superior BPDU** 会触发 spanning-tree 重新计算，让攻击者成为 root bridge，从而实施**中间人攻击**。
- **BPDU Guard** 会把收到 BPDU 的接口置为 **err-disable**（`BLOCK_BPDUGUARD`、`ERR_DISABLE` 消息）。
- 接口：`spanning-tree bpduguard enable` / 全局：`spanning-tree portfast bpduguard default`（所有 PortFast 端口）。
- 用在**连接计算机的 access 端口**上，而不是交换机之间的链路上。处于 err-disable 的端口在排除原因后用 `shutdown` / `no shutdown` 恢复。
- 用 `show spanning-tree summary` 查看全局 PortFast 和 BPDU Guard 设置。
