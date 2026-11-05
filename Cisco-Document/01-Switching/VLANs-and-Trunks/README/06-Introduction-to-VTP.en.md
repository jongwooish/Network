# Introduction to VLAN Trunking Protocol (VTP)

[한국어](../06-Introduction-to-VTP.md) | **English** | [日本語](06-Introduction-to-VTP.ja.md) | [简体中文](06-Introduction-to-VTP.zh-CN.md) | [Español](06-Introduction-to-VTP.es.md) | [Français](06-Introduction-to-VTP.fr.md)

If you have 20 switches and 50 VLANs, creating every VLAN on every switch by hand takes far too long. With **VTP (VLAN Trunking Protocol)**, you create VLANs on one switch and the other switches receive that information and **synchronize automatically**.

- A Cisco-proprietary protocol
- VTP information travels **only over trunk links**
- Only switches with the same **VTP domain name** synchronize with each other

---

## 1. How VTP works

![VTP domain](../images/vtp-domain.svg)

1. Add / modify / delete VLANs on the **VTP Server**.
2. Every change increases the **Configuration Revision** number by 1.
3. The latest information (advertisement) is sent to all VTP Clients.
4. A VTP Client synchronizes when it receives information with a **higher revision than its own**.

> A VTP Server also synchronizes (it acts as a client too). So you can have several VTP Servers and change VLANs on any of them.

---

## 2. VTP modes

### VTP Transparent

![VTP transparent](../images/vtp-transparent.svg)

A VTP Transparent switch **only forwards** advertisements to other switches and **does not synchronize itself**. Instead, it can create VLANs **locally**.

When you create VLAN 20 on the VTP Server:
1. VLAN 20 is created on the VTP Server
2. The revision number increases
3. The VTP Server sends the advertisement to the Transparent switch
4. The Transparent switch does not synchronize, but **forwards** the advertisement to the Client
5. The Client synchronizes with the latest information

### Mode comparison

| | **VTP Server** | **VTP Client** | **VTP Transparent** |
| --- | --- | --- | --- |
| Create / modify / delete VLANs | ✅ | ❌ | Locally only |
| Synchronizes itself | ✅ | ✅ | ❌ |
| Forwards advertisements | ✅ | ✅ | ✅ |
| Where VLAN info is stored | `vlan.dat` | `vlan.dat` | **running-config** (+ `vlan.dat`) |

> There is also an **`off` mode** that disables VTP entirely. It is like Transparent but **does not forward** advertisements either (VTP version 3, or IOS versions that support it).

---

## 3. The danger of VTP: the revision number

VTP is convenient but has a serious risk. **A VTP Server also synchronizes like a Client**, and every switch follows the information with the **highest revision number**.

This can happen:

1. You take a VTP Client switch out of the production network and bring it to a lab
2. You change it to VTP Server mode
3. You create and modify VLANs while testing → **revision increases** with every change
4. When done, you **delete all the VLANs**
5. You change it back to VTP Client mode
6. You **reconnect** that switch to the production network

Because this switch's revision is **higher** than the production network's, the other switches synchronize to it → **all production VLANs disappear.** Ports that belonged to those VLANs don't even return to VLAN 1; they stay in VLANs that no longer exist, and **traffic stops.**

> **Prevention**
> - Before connecting a switch that was used elsewhere, **reset its revision to 0**: temporarily change the VTP domain name and change it back, or switch to Transparent mode and back.
> - Set a domain password with `vtp password`.
> - With **VTP version 3**, only the switch designated as the **Primary Server** can change VLAN information, which prevents this accident.
> - Many networks that don't need VTP simply run in **Transparent** (or `off`) mode.

---

## 4. VTP pruning

![VTP pruning](../images/vtp-pruning.svg)

There are PCs in VLANs 10, 20 and 30, and the links between switches are **802.1Q trunks** carrying all VLANs.
When a PC in VLAN 10 sends a **broadcast**, switches flood it and the trunks carry every VLAN, so it **reaches every switch.**

But the switch in the middle has **no VLAN 10 PCs at all** (only VLAN 20). That broadcast is **wasted bandwidth**.

With **VTP pruning** enabled, traffic for a VLAN is not sent down trunks toward switches that have no devices in that VLAN.

```
Switch(config)#vtp pruning
```

- Enabling it on a Server applies it to the whole domain.
- Depending on the switch model, it is on or off by default.
- VLAN 1 and VLANs 1002 ~ 1005 are never pruned.

---

## 5. VTP configuration lab

![VTP lab topology](../images/vtp-lab-topology.svg)

Three switches (SW1, SW2, SW3) are connected with trunks. The VLAN database and startup-config have been erased on all switches.

### 5-1. Check the default state

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

SW2 and SW3 look the same.

| Field | Meaning |
| --- | --- |
| `Configuration Revision : 0` | Increases every time VLANs are added or removed. Still 0 because nothing has changed |
| `VTP Operating Mode : Server` | The default mode is **Server** |
| `VTP Pruning Mode : Disabled` | Pruning is off |
| `VTP version running : 1` | Supports versions 1–3, but currently runs version 1 |
| `VTP Domain Name :` (blank) | No domain name yet |

### 5-2. Create a VLAN on SW1

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

SW1's revision went up to **1**. But SW2 and SW3 are still at **0** and don't have VLAN 10.
→ Because no **VTP domain name** has been configured.

### 5-3. Configure the VTP domain

Turn on debugging on SW2 and SW3 to watch in real time.

```
SW2#debug sw-vlan vtp events
vtp events debugging is on
```

Set the domain name on SW1.

```
SW1(config)#vtp domain JJONG
Changing VTP domain name from NULL to JJONG
```

Debug output on SW3:

```
VTP LOG RUNTIME: Summary packet received in NULL domain state
VTP LOG RUNTIME: Summary packet received, domain = JJONG, rev = 1, followers = 1, length 77, trunk Gi0/0
VTP LOG RUNTIME: Transitioning from NULL to JJONG domain
VTP LOG RUNTIME: Summary packet rev 1 greater than domain JJONG rev 0
```

- A switch with **no domain name (NULL)** **adopts** the domain name from the received VTP packet (`JJONG`). (It won't change it if it already has a domain name.)
- The received packet's revision (**1**) is higher than its own (**0**), so it **synchronizes.**

Debug output is noisy, so always turn it off.

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

SW2 and SW3 have **learned VLAN 10** through VTP.

### 5-4. Create VLANs on several Servers

All three switches are Servers right now, so VLANs can be created on any of them.

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

SW2 and SW3 also have 10, 20 and 30, and the revision is **3** on all three switches. It goes up by 1 with each new VLAN.

### 5-5. VTP Client mode

```
SW2(config)#vtp mode client
Setting device to VTP Client mode for VLANS.
```

```
SW2#show vtp status | include Operating
VTP Operating Mode                : Client
```

Now SW1 and SW3 are Servers and SW2 is a Client. For this test, **disconnect the SW1 ↔ SW3 link** so there is no direct connection between them. (SW1 — SW2 — SW3)

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

SW3 learns VLAN 40 too, via SW1 → **SW2 (Client)** → SW3.
→ **A Client synchronizes itself and also forwards advertisements to other switches.**

VLANs can't be created on a Client.

```
SW2(config)#vlan 50
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

### 5-6. VTP Transparent mode

With the SW1 ↔ SW3 link still disconnected, change SW2 to Transparent.

```
SW2(config)#vtp mode transparent
Setting device to VTP Transparent mode for VLANS.
```

Create VLAN 50 on SW1.

```
SW1(config)#vlan 50
SW1(config-vlan)#name Research
```

| Switch | VLAN 50 | Why |
| --- | --- | --- |
| SW1 (Server) | ✅ Present | Created there |
| SW2 (Transparent) | ❌ Missing | Transparent **doesn't synchronize** |
| SW3 (Server) | ✅ Present | SW2 **forwarded** the advertisement |

Now create VLAN 60 on SW2 (Transparent).

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

A Transparent switch can create VLANs **locally**. But VLAN 60 **does not appear** on SW1 or SW3, because a Transparent switch's VLANs are local only and are not advertised.

### 5-7. Transparent stores VLANs in the running-config

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

| Mode | Where VLAN info is stored |
| --- | --- |
| Server / Client | `flash:vlan.dat` (not shown in the running-config) |
| **Transparent** | Also in the **running-config** → save with `copy run start` |

---

## 6. Command summary

| Command | Description |
| --- | --- |
| `vtp domain <name>` | Set the VTP domain name (case-sensitive) |
| `vtp mode {server \| client \| transparent \| off}` | Change the VTP mode |
| `vtp version {1 \| 2 \| 3}` | Change the VTP version |
| `vtp password <password>` | Set the domain password |
| `vtp pruning` | Enable VTP pruning |
| `show vtp status` | Check version, domain, mode and revision |
| `show vtp password` | Show the configured password |
| `debug sw-vlan vtp events` | Watch VTP events in real time (turn off with `no debug all`) |

---

## 7. Key takeaways

- VTP **automatically synchronizes** VLANs created on one switch to the switches in the same domain. It only works when the **domain names match**, and it travels only over **trunks**.
- **Server**: can create VLANs + synchronizes + forwards / **Client**: synchronizes + forwards / **Transparent**: forwards only, its VLANs are local
- Every switch follows the information with the **highest revision**. Reset a used switch's **revision to 0** before connecting it, or you risk wiping every VLAN.
- **VTP pruning** saves bandwidth by blocking VLAN traffic on trunks where it isn't needed.
- Transparent stores VLANs in the **running-config**; Server/Client store them in **vlan.dat**.
