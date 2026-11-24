# Spanning-Tree Root Guard

[한국어](../02-Root-Guard.md) | **English** | [日本語](02-Root-Guard.ja.md) | [简体中文](02-Root-Guard.zh-CN.md) | [Español](02-Root-Guard.es.md) | [Français](02-Root-Guard.fr.md)

**Root Guard** makes sure you don't accept a certain switch as the root bridge. BPDUs are sent and processed normally, but if a switch suddenly sends a BPDU with a **superior bridge ID**, you won't accept that switch as the root bridge. Normally the switch with the best bridge ID would become the root bridge, but that won't happen on a port with Root Guard.

## Root Guard in a nutshell

| Situation | Behavior |
| --- | --- |
| Normally | BPDUs are sent and received as usual. |
| Superior BPDU received on a Root Guard port | The port is blocked as **root-inconsistent** for that VLAN. |
| Superior BPDUs stop arriving | The port recovers automatically. |

---

## 1. Lab topology

We'll use this topology.

![Lab topology: VPC, SW2, SW3](../images/root-guard/01-topology.png)

The configuration uses SW2 and SW3. First, make sure SW3 is **not** the root bridge by lowering SW2's priority so that SW2 becomes the root bridge.

![spanning-tree vlan 1 priority 4096 on SW2](../images/root-guard/02-sw2-priority.png)

---

## 2. Configuring Root Guard

Enable Root Guard on SW2's interface toward SW3.

![spanning-tree guard root on Gi0/0](../images/root-guard/03-root-guard.png)

We get a message (`ROOTGUARD_CONFIG_CHANGE`) that Root Guard has been enabled.

> The original uses Fastethernet0/16 and the topology picture labels the SW2–SW3 link as Gi0/1, but in the lab output the port toward SW3 is **Gi0/0**. This page follows the lab output.

Let's enable a debug to see what's going on.

![debug spanning-tree events on SW2](../images/root-guard/04-debug.png)

---

## 3. What happens with a superior BPDU?

Now let's upset SW2 by changing SW3's priority to the lowest possible value (0). Normally SW3 should now become the root bridge.

![spanning-tree vlan 1 priority 0 on SW3](../images/root-guard/05-sw3-priority.png)

What does SW2 think about this?

![ROOTGUARD_BLOCK on SW2](../images/root-guard/06-root-guard-block.png)

SW2 hears a better root (`1-505b.2e00.0200`) on Gi0/0, but it **won't accept** SW3 as the root bridge. Instead it blocks the interface for this VLAN (`ROOTGUARD_BLOCK`). Here's another useful command to verify this:

![show spanning-tree inconsistentports](../images/root-guard/07-inconsistentports.png)

It tells us that GigabitEthernet0/0 is **Root Inconsistent** for VLAN 1.

Root Guard is useful on your **core or distribution layer switches**, so the switches below them can never be elected root bridge.

---

## Summary

- **Root Guard** refuses to accept a new root bridge based on a superior BPDU received on the port, and blocks the port as **root-inconsistent**.
- Configure it with `spanning-tree guard root` on the interface. You'll see `ROOTGUARD_CONFIG_CHANGE` when it's enabled and `ROOTGUARD_BLOCK` when it blocks.
- `show spanning-tree inconsistentports` shows the blocked port and VLAN (`Root Inconsistent`).
- Enable it on core/distribution switches on the **ports facing the lower (access) switches** to protect the root bridge location.
