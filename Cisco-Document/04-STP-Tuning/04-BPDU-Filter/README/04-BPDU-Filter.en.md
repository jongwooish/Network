# Spanning-Tree BPDU Filter

[한국어](../04-BPDU-Filter.md) | **English** | [日本語](04-BPDU-Filter.ja.md) | [简体中文](04-BPDU-Filter.zh-CN.md) | [Español](04-BPDU-Filter.es.md) | [Français](04-BPDU-Filter.fr.md)

The **BPDU Filter** works much like [BPDU Guard](../../03-BPDU-Guard/README/03-BPDU-Guard.en.md) in that it lets you block malicious BPDUs. The difference is that BPDU Guard puts the interface that receives the BPDU into **err-disable**, while BPDU Filter just **filters** it. In this lesson we'll take a good look at how BPDU Filter works.

## Global vs interface configuration

BPDU Filter can be configured globally or on the interface, and they behave differently.

| Where | Command | Behavior |
| --- | --- | --- |
| Global | `spanning-tree portfast bpdufilter default` | **Interfaces with PortFast enabled** neither send nor receive BPDUs. But when a BPDU arrives on such an interface, it **loses its PortFast status, BPDU Filter is disabled**, and it acts as a normal interface. |
| Interface | `spanning-tree bpdufilter enable` | Incoming BPDUs are **ignored**, and **no BPDUs are sent**. This is effectively **disabling spanning tree** on that interface. |

Be careful when you enable BPDU Filter on interfaces. You can use it on access-mode interfaces connected to computers, but **never configure it on interfaces connected to other switches**; if you do, you might end up with a loop.

---

## 1. Lab topology

We'll use this topology to demonstrate BPDU Filter.

![Lab topology: SW2, SW3](../images/bpdu-filter/01-topology.png)

---

## 2. Configuring it on an interface

Enable PortFast on SW2's interface, then enable BPDU Filter.

![spanning-tree portfast trunk, spanning-tree bpdufilter enable](../images/bpdu-filter/02-bpdufilter.png)

The lab link is a trunk, so PortFast was enabled with `spanning-tree portfast trunk`. SW2 now stops sending BPDUs on this interface and ignores whatever it receives. Let's enable a debug to see what it does.

![debug spanning-tree bpdu on SW2](../images/bpdu-filter/03-debug.png)

With BPDU debugging enabled, **you won't see anything**, because SW2 no longer sends or processes BPDUs. If you want, enable BPDU debugging on SW3 too; you'll see that it doesn't receive any from SW2. Let's remove the interface command.

![no spanning-tree bpdufilter enable](../images/bpdu-filter/04-remove.png)

---

## 3. Configuring it globally

Now let's enable BPDU Filter globally.

```
SW2(config)#spanning-tree portfast bpdufilter default
```

> In the PDF, the capture for this step is the same screen as the previous `no spanning-tree bpdufilter enable` capture, so the global command is shown as a code block.

This enables BPDU Filter on **all interfaces with PortFast enabled**.

That's all there is to it. It's probably **better to use BPDU Guard**, though. If you don't expect BPDUs on an interface, it's better to **get a notification** through err-disable than to filter them silently and never see what's going on.

---

## Summary

- **BPDU Filter** **filters** BPDUs without err-disable; BPDU Guard, by contrast, err-disables the port.
- **Global** (`spanning-tree portfast bpdufilter default`): PortFast ports don't send or receive BPDUs, but receiving one removes PortFast and BPDU Filter and the port becomes normal.
- **Interface** (`spanning-tree bpdufilter enable`): ignores BPDUs and sends none — effectively **disabling spanning tree** on that port.
- Using it on ports connected to other switches can cause **loops**. Use it only on host ports, and prefer **BPDU Guard**, which notifies you.
