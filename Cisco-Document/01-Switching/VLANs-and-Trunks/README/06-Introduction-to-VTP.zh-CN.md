# Introduction to VLAN Trunking Protocol (VTP)

[한국어](../06-Introduction-to-VTP.md) | [English](06-Introduction-to-VTP.en.md) | [日本語](06-Introduction-to-VTP.ja.md) | **简体中文** | [Español](06-Introduction-to-VTP.es.md) | [Français](06-Introduction-to-VTP.fr.md)

如果有 20 台交换机和 50 个 VLAN，在每台交换机上逐个手动创建 VLAN 会非常耗时。使用 **VTP (VLAN Trunking Protocol)**，只需在一台交换机上创建 VLAN，其他交换机就会收到这些信息并**自动同步**。

- Cisco 私有协议
- VTP 信息**只通过 Trunk 链路**传递
- 只有 **VTP 域名**相同的交换机之间才会同步

---

## 1. VTP 的工作方式

![VTP domain](../images/vtp-domain.svg)

1. 在 **VTP Server** 上添加 / 修改 / 删除 VLAN。
2. 每次变更，**Configuration Revision** 编号加 1。
3. 最新信息(advertisement)发送给所有 VTP Client。
4. VTP Client 收到 **revision 比自己高**的信息时，就按该内容同步。

> VTP Server 也会同步(同时承担 Client 的角色)。因此可以部署多台 VTP Server，在任意一台上修改 VLAN。

---

## 2. VTP 模式

### VTP Transparent

![VTP transparent](../images/vtp-transparent.svg)

VTP Transparent **只把** advertisement **转发**给其他交换机，**自己不同步**。但它可以**只在本机**创建 VLAN。

在 VTP Server 上创建 VLAN 20 时:
1. 在 VTP Server 上创建 VLAN 20
2. revision 编号增加
3. VTP Server 把 advertisement 发给 Transparent 交换机
4. Transparent 不同步，而是把 advertisement **转发**给 Client
5. Client 按最新信息同步

### 模式对比

| | **VTP Server** | **VTP Client** | **VTP Transparent** |
| --- | --- | --- | --- |
| 创建 / 修改 / 删除 VLAN | ✅ | ❌ | 仅限本机 |
| 自身同步 | ✅ | ✅ | ❌ |
| 转发 advertisement | ✅ | ✅ | ✅ |
| VLAN 信息保存位置 | `vlan.dat` | `vlan.dat` | **running-config** (+ `vlan.dat`) |

> 此外还有完全关闭 VTP 的 **`off` 模式**。它与 Transparent 类似，但**连 advertisement 也不转发**(VTP 版本 3，或支持该模式的 IOS)。

---

## 3. VTP 的风险: revision 编号

VTP 很方便，但有很大的风险。**VTP Server 也会像 Client 一样同步**，而所有交换机都会跟随 **revision 编号最高的信息**。

可能会发生这样的情况:

1. 从生产网络中取下一台 VTP Client 交换机，带到实验室
2. 把它改成 VTP Server 模式
3. 创建、修改 VLAN 进行各种测试 → 每次变更 **revision 都会增加**
4. 测试结束后**删除所有 VLAN**
5. 再改回 VTP Client 模式
6. 把这台交换机**重新接入生产网络**

由于这台交换机的 revision **高于**生产网络，其他交换机会按它的信息同步 → **生产环境的 VLAN 全部消失。** 原本属于这些 VLAN 的端口也不会回到 VLAN 1，而是留在已不存在的 VLAN 中，**通信中断。**

> **预防方法**
> - 接入在其他地方用过的交换机之前，先**把 revision 重置为 0**: 临时把 VTP 域名改成别的名称再改回来，或者切换到 Transparent 模式再切回来。
> - 用 `vtp password` 为域设置密码。
> - 使用 **VTP 版本 3** 时，只有被指定为 **Primary Server** 的交换机才能修改 VLAN 信息，可以防止这种事故。
> - 很多不需要 VTP 的网络直接以 **Transparent**(或 `off`)模式运行。

---

## 4. VTP 修剪 (Pruning)

![VTP pruning](../images/vtp-pruning.svg)

VLAN 10、20、30 中都有 PC，交换机之间是承载所有 VLAN 的 **802.1Q Trunk**。
VLAN 10 的 PC 发出**广播**时，交换机会泛洪，而 Trunk 承载所有 VLAN，因此广播会**传到所有交换机。**

但中间那台交换机上**没有任何** VLAN 10 的 PC(只有 VLAN 20)。这个广播就是**浪费带宽**。

启用 **VTP Pruning** 后，不会再向没有该 VLAN 设备的那一侧 Trunk 发送该 VLAN 的流量。

```
Switch(config)#vtp pruning
```

- 在 Server 上启用后，会应用到整个域。
- 根据交换机型号，默认值为 On 或 Off。
- VLAN 1 和 1002 ~ 1005 不会被修剪。

---

## 5. VTP 配置实验

![VTP lab topology](../images/vtp-lab-topology.svg)

用 Trunk 连接 3 台交换机(SW1、SW2、SW3)，在清除所有交换机的 VLAN 数据库和 startup-config 的状态下开始。

### 5-1. 查看默认状态

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

SW2、SW3 也一样。

| 项目 | 含义 |
| --- | --- |
| `Configuration Revision : 0` | 每次添加或删除 VLAN 都会增加。还没有任何变更，所以是 0 |
| `VTP Operating Mode : Server` | 默认模式是 **Server** |
| `VTP Pruning Mode : Disabled` | 修剪已关闭 |
| `VTP version running : 1` | 支持版本 1~3，但当前以版本 1 运行 |
| `VTP Domain Name :` (空白) | 还没有域名 |

### 5-2. 在 SW1 上创建 VLAN

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

SW1 的 revision 变成了 **1**。但 SW2、SW3 仍然是 **0**，也没有 VLAN 10。
→ 因为还没有配置 **VTP 域名**。

### 5-3. 配置 VTP 域

为了实时观察，在 SW2、SW3 上打开 debug。

```
SW2#debug sw-vlan vtp events
vtp events debugging is on
```

在 SW1 上设置域名。

```
SW1(config)#vtp domain JJONG
Changing VTP domain name from NULL to JJONG
```

SW3 的 debug 输出:

```
VTP LOG RUNTIME: Summary packet received in NULL domain state
VTP LOG RUNTIME: Summary packet received, domain = JJONG, rev = 1, followers = 1, length 77, trunk Gi0/0
VTP LOG RUNTIME: Transitioning from NULL to JJONG domain
VTP LOG RUNTIME: Summary packet rev 1 greater than domain JJONG rev 0
```

- **没有域名(NULL)**的交换机会把收到的 VTP 报文中的域名(`JJONG`)**设为自己的域名。**(如果已经有域名则不会更改。)
- 收到报文的 revision(**1**)比自己的(**0**)高，所以会**同步。**

debug 输出很多，一定要记得关闭。

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

SW2、SW3 通过 VTP **学到了 VLAN 10。**

### 5-4. 在多台 Server 上创建 VLAN

现在三台交换机都是 Server，所以可以在任意一台上创建 VLAN。

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

SW2、SW3 同样拥有 10、20、30，三台交换机的 revision 都是 **3**。每创建一个 VLAN 加 1。

### 5-5. VTP Client 模式

```
SW2(config)#vtp mode client
Setting device to VTP Client mode for VLANS.
```

```
SW2#show vtp status | include Operating
VTP Operating Mode                : Client
```

现在 SW1、SW3 是 Server，SW2 是 Client。为了测试，**断开 SW1 ↔ SW3 的链路**，让两者之间没有直接连接。(SW1 — SW2 — SW3)

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

SW3 也学到了 VLAN 40，路径是 SW1 → **SW2(Client)** → SW3。
→ **Client 也会自身同步，并把 advertisement 转发给其他交换机。**

Client 上不能创建 VLAN。

```
SW2(config)#vlan 50
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

### 5-6. VTP Transparent 模式

在 SW1 ↔ SW3 链路保持断开的情况下，把 SW2 改为 Transparent。

```
SW2(config)#vtp mode transparent
Setting device to VTP Transparent mode for VLANS.
```

在 SW1 上创建 VLAN 50。

```
SW1(config)#vlan 50
SW1(config-vlan)#name Research
```

| 交换机 | VLAN 50 | 原因 |
| --- | --- | --- |
| SW1 (Server) | ✅ 有 | 直接创建 |
| SW2 (Transparent) | ❌ 没有 | Transparent **不同步** |
| SW3 (Server) | ✅ 有 | SW2 **转发**了 advertisement |

接下来在 SW2(Transparent)上创建 VLAN 60。

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

Transparent 可以**在本机**创建 VLAN。但 SW1、SW3 上**看不到** VLAN 60，因为 Transparent 的 VLAN 只在本机使用，不会被通告。

### 5-7. Transparent 把 VLAN 保存在 running-config 中

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

| 模式 | VLAN 信息保存位置 |
| --- | --- |
| Server / Client | `flash:vlan.dat` (不会出现在 running-config 中) |
| **Transparent** | 也会出现在 **running-config** 中 → 用 `copy run start` 保存 |

---

## 6. 命令汇总

| 命令 | 说明 |
| --- | --- |
| `vtp domain <名称>` | 设置 VTP 域名 (区分大小写) |
| `vtp mode {server \| client \| transparent \| off}` | 更改 VTP 模式 |
| `vtp version {1 \| 2 \| 3}` | 更改 VTP 版本 |
| `vtp password <密码>` | 设置域密码 |
| `vtp pruning` | 启用 VTP 修剪 |
| `show vtp status` | 查看版本、域、模式、revision |
| `show vtp password` | 查看已设置的密码 |
| `debug sw-vlan vtp events` | 实时查看 VTP 事件 (用 `no debug all` 关闭) |

---

## 7. 要点总结

- VTP 会把在一台交换机上创建的 VLAN **自动同步到同一域内的交换机**。**域名必须相同**才能工作，并且只通过 **Trunk** 传递。
- **Server**: 可创建 VLAN + 同步 + 转发 / **Client**: 同步 + 转发 / **Transparent**: 只转发，VLAN 只属于本机
- 所有交换机都跟随 **revision 最高**的信息。在别处用过的交换机接入前，必须把 **revision 置为 0**，才能避免所有 VLAN 被删除的事故。
- **VTP 修剪**通过阻止 VLAN 流量进入不需要的 Trunk 来节省带宽。
- Transparent 把 VLAN 保存在 **running-config** 中，Server/Client 保存在 **vlan.dat** 中。
