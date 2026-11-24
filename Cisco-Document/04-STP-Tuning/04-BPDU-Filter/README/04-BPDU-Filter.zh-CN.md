# Spanning-Tree BPDU Filter

[한국어](../04-BPDU-Filter.md) | [English](04-BPDU-Filter.en.md) | [日本語](04-BPDU-Filter.ja.md) | **简体中文** | [Español](04-BPDU-Filter.es.md) | [Français](04-BPDU-Filter.fr.md)

**BPDU Filter** 与 [BPDU Guard](../../03-BPDU-Guard/README/03-BPDU-Guard.zh-CN.md) 类似，也可以用来阻挡恶意 BPDU。区别在于 BPDU Guard 会把收到 BPDU 的接口置为 **err-disable**，而 BPDU Filter 只是把 BPDU **过滤（filter）** 掉。本文详细介绍 BPDU Filter 的工作方式。

## 全局配置与接口配置的区别

BPDU Filter 可以在全局或接口上配置，两者的行为不同。

| 配置位置 | 命令 | 行为 |
| --- | --- | --- |
| 全局 | `spanning-tree portfast bpdufilter default` | **启用了 PortFast 的接口**既不发送也不接收 BPDU。但如果这样的接口收到 BPDU，它会**失去 PortFast 状态，BPDU Filter 也随之关闭**，变成普通接口。 |
| 接口 | `spanning-tree bpdufilter enable` | **忽略**收到的 BPDU，也**不发送** BPDU。这实际上等同于在该接口上**关闭 spanning-tree**。 |

在接口上启用 BPDU Filter 时要小心。可以用在连接计算机的 access 模式接口上，但**绝不要配置在连接其他交换机的接口上**，否则可能出现环路。

---

## 1. 实验拓扑

用下面的拓扑演示 BPDU Filter。

![Lab topology: SW2, SW3](../images/bpdu-filter/01-topology.png)

---

## 2. 在接口上配置

在 SW2 的接口上启用 PortFast，然后启用 BPDU Filter。

![spanning-tree portfast trunk, spanning-tree bpdufilter enable](../images/bpdu-filter/02-bpdufilter.png)

实验链路是 trunk，所以用 `spanning-tree portfast trunk` 启用 PortFast。现在 SW2 不再从这个接口发送 BPDU，收到的 BPDU 也会被忽略。用 debug 看看它的表现。

![debug spanning-tree bpdu on SW2](../images/bpdu-filter/03-debug.png)

开启 BPDU debug 后**什么也看不到**，因为 SW2 已经不再发送或处理 BPDU。如果愿意，也可以在 SW3 上开启 BPDU debug，你会发现收不到任何来自 SW2 的 BPDU。删除接口命令。

![no spanning-tree bpdufilter enable](../images/bpdu-filter/04-remove.png)

---

## 3. 全局配置

这次全局启用 BPDU Filter。

```
SW2(config)#spanning-tree portfast bpdufilter default
```

> PDF 中这一步的截图与前一步 `no spanning-tree bpdufilter enable` 的截图是同一画面，所以这里用代码块写出全局命令。

这条命令会在**所有启用了 PortFast 的接口**上启用 BPDU Filter。

就是这些。不过通常**还是用 BPDU Guard 更好**。如果某个接口本不应该收到 BPDU，与其悄悄过滤掉而不知道发生了什么，不如通过 err-disable **收到通知**。

---

## 总结

- **BPDU Filter** 不触发 err-disable，只是**过滤** BPDU；而 BPDU Guard 会把端口置为 err-disable。
- **全局**（`spanning-tree portfast bpdufilter default`）：PortFast 端口不收发 BPDU，但一旦收到 BPDU，PortFast 和 BPDU Filter 就会解除，端口变为普通端口。
- **接口**（`spanning-tree bpdufilter enable`）：忽略 BPDU 且不发送，实际上就是**关闭**该端口的 **spanning-tree**。
- 用在连接其他交换机的端口上可能造成**环路**。只用在主机端口上，并尽量使用能发出通知的 **BPDU Guard**。
