# Spanning-Tree Port States

[한국어](../03-Spanning-Tree-Port-States.md) | **English** | [日本語](03-Spanning-Tree-Port-States.ja.md) | [简体中文](03-Spanning-Tree-Port-States.zh-CN.md) | [Español](03-Spanning-Tree-Port-States.es.md) | [Français](03-Spanning-Tree-Port-States.fr.md)

When you plug a cable into a Cisco switch, the LED above the interface is **orange** at first and turns **green** after a while. During that time, spanning-tree is deciding the state of the interface. This note covers each port state and shows the states changing on a real switch.

> For the basics of spanning-tree, read [01. Introduction to Spanning-Tree](01-Introduction-to-Spanning-Tree.en.md) first.

---

## 1. Port states

When you plug in a cable, this is what happens:

- **Listening**: only a root port or designated port moves to the listening state. A non-designated port stays in the blocking state. No data is transmitted for **15 seconds**, to make sure the topology doesn't change in the meantime. Then the port moves to learning.
- **Learning**: the interface processes Ethernet frames by looking at the **source MAC address** to fill the MAC address table. However, frames are not forwarded to their destination. After **15 seconds** it moves to forwarding.
- **Forwarding**: the final state. The interface finally forwards Ethernet frames, so **data is transmitted**.

A port that is not a designated or root port is in the **blocking** state.

So it takes **30 seconds in total** to go from listening to forwarding. Not really fast, right? And this happens on every interface of the switch.

Sometimes an interface in blocking mode has to move to forwarding because the topology changed. In that case, the blocking state lasts **20 seconds** before it moves to listening. So it takes 20 (blocking) + 15 (listening) + 15 (learning) = **50 seconds** before the interface is forwarding.

Thirty seconds is a long time. A modern PC with an SSD boots faster than that. Here is an overview of the port states:

| State | Forward frames | Learn MAC addresses | Duration |
| --- | --- | --- | --- |
| Blocking | No | No | 20 seconds |
| Listening | No | No | 15 seconds |
| Learning | No | Yes | 15 seconds |
| Forwarding | Yes | Yes | – |

---

## 2. Checking on a real switch

What does this look like on a real Cisco switch? I unplugged and plugged in the cable of an interface (or did a shutdown and no shutdown) and ran the show command right away:

![Listening state](../images/port-states/01-listening.png)

The role of the port is **designated** and the status is **listening (LIS)**. Keep running the show command, and after about 15 seconds it looks like this:

![Learning state](../images/port-states/02-learning.png)

It has moved to the **learning (LRN)** state. After another 15 seconds or so, it looks like this:

![Forwarding state](../images/port-states/03-forwarding.png)

You just watched an interface go through the port states to **forwarding (FWD)**. A better way to see the changes is to enable a **debug**.

![debug spanning-tree events](../images/port-states/04-debug-on.png)

When we disable and enable the interface again, we can see the port states change in real time.

![Debug output](../images/port-states/05-debug-output.png)

The state changes `listening` → `learning` → `forwarding`, with about **15 seconds** between each step.

---

## Key points

- **Listening (15 s)**: only BPDUs are exchanged; no MAC learning and no data transmission.
- **Learning (15 s)**: the MAC address table is filled, but frames are not forwarded yet.
- **Forwarding**: frames are forwarded. Ports that are not root or designated stay in **blocking**.
- After plugging in a cable it takes **30 seconds** to forward, and **50 seconds** (20 + 15 + 15) when starting from blocking.
- Check the state changes by repeating `show spanning-tree` or with `debug spanning-tree events`.
