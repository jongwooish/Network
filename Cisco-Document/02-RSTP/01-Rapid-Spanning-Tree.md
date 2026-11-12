# Rapid Spanning-Tree (RSTP)

**한국어** | [English](README/01-Rapid-Spanning-Tree.en.md) | [日本語](README/01-Rapid-Spanning-Tree.ja.md) | [简体中文](README/01-Rapid-Spanning-Tree.zh-CN.md) | [Español](README/01-Rapid-Spanning-Tree.es.md) | [Français](README/01-Rapid-Spanning-Tree.fr.md)

요즘 네트워크에는 라우팅이 점점 많아지고 있습니다. OSPF, EIGRP 같은 라우팅 프로토콜은 네트워크 변화에 Spanning Tree보다 훨씬 빠르게 대응합니다. 이 속도를 따라가기 위해 만들어진 것이 **Rapid Spanning Tree (RSTP, IEEE 802.1w)** 입니다.

RSTP는 기존 Spanning Tree의 **혁명이 아니라 진화**입니다. 속도를 높이려고 내부 동작 몇 가지를 바꿨을 뿐, **설정 방법은 그대로**입니다. 아래에서는 기존 Spanning Tree(802.1D)를 **Classic Spanning Tree** 라고 부르겠습니다.

## 한눈에 보는 차이

| 항목 | Classic STP (802.1D) | RSTP (802.1w) |
| --- | --- | --- |
| Port State | Blocking / Listening / Learning / Forwarding | **Discarding** / Learning / Forwarding |
| Port Role | Root, Designated, Alternate(Blocked) | Root, Designated, Alternate, **Backup** |
| BPDU | Flags 2비트만 사용 (TC, TCA) | **Version 2 BPDU**, Flags 전부 사용 |
| BPDU 생성 | Root만 생성, 나머지는 중계 | **모든 스위치가 hello time(2초)마다** 생성 |
| 장애 감지 | Max Age 20초 | BPDU **3개 놓치면** 바로 감지 (keepalive) |
| Inferior BPDU | 무시 | **받아들임** (BackboneFast와 비슷) |
| Forwarding 전환 | 타이머 기반 (Listening → Learning, 약 30초) | **Proposal / Agreement 협상 (Sync)** |
| PortFast | PortFast | **Edge port** |
| UplinkFast | 직접 설정 | **기본 내장** |
| Topology Change | Root까지 TCN 전송 → Root가 알림 | **감지한 스위치가 바로 Flooding** |

---

## 1. Port State: 4개 → 3개

![Port states](images/rstp-port-states.svg)

RSTP의 Port State는 **Discarding, Learning, Forwarding** 세 가지뿐입니다. **Discarding** 은 Classic의 Blocking과 Listening을 합친 상태입니다.

| Classic Spanning Tree | Rapid Spanning Tree | 토폴로지에서 활성? | MAC 주소 학습? |
| --- | --- | --- | --- |
| Blocking | Discarding | No | No |
| Listening | Discarding | Yes | No |
| Learning | Learning | Yes | Yes |
| Forwarding | Forwarding | Yes | Yes |

---

## 2. Port Role

![Port roles](images/rstp-port-roles.svg)

- **Root Bridge**: Bridge ID(Priority + MAC)가 가장 좋은 스위치. 여기서는 SW1(MAC AAA).
- **Root port**: Non-root 스위치에서 Root까지 비용이 가장 낮은 포트.
- **Designated port**: 세그먼트마다 하나만 존재. **가장 좋은 BPDU를 보낼 수 있는** 포트가 됩니다.
- **Alternate port**: SW3의 Fa0/16은 SW2에게서 더 좋은 BPDU를 받으므로 막힙니다. Root port의 예비 경로입니다.

여기까지는 Classic Spanning Tree와 똑같습니다. RSTP에서 새로 생긴 역할은 **Backup port** 입니다.

### Backup port

![Backup port](images/rstp-backup-port.svg)

SW2와 SW3 사이에 **Hub** 를 넣었습니다. Hub가 없다면 SW2의 Fa0/16, Fa0/17은 둘 다 Designated port가 됩니다.

하지만 Hub 때문에 SW2의 Fa0/16과 Fa0/17이 **같은 collision domain** 에 묶였습니다. SW2는 자기가 보낸 BPDU를 Fa0/16, Fa0/17 둘 다에서 다시 받기 때문에, 같은 세그먼트에 연결이 두 개 있다는 걸 알게 됩니다.

- Fa0/16 → **Designated port**
- Fa0/17 → Fa0/16의 **Backup port**

> 실무 네트워크에서 Backup port를 볼 일은 거의 없습니다. Hub를 빼면 다시 둘 다 Designated port가 됩니다.

---

## 3. BPDU (Version 2)

![RSTP BPDU](images/rstp-bpdu-flags.svg)

Classic Spanning Tree는 Flags 필드에서 **두 비트만** 썼습니다.
- Topology Change
- Topology Change Acknowledgement

RSTP는 **Flags 필드를 전부** 사용합니다. 특히 **Port Role** 필드로 BPDU를 보낸 포트의 역할을 알려 줍니다.
- Unknown
- Alternate / Backup port
- Root port
- Designated port

이 새 BPDU를 **Version 2 BPDU** 라고 부릅니다. Classic Spanning Tree만 쓰는 스위치는 이 BPDU를 버립니다.

---

## 4. 그 밖에 바뀐 것들

### BPDU는 모든 스위치가 hello time마다 보낸다
Classic에서는 **Root Bridge만** BPDU를 만들고, Non-root 스위치는 Root port로 받은 BPDU를 중계만 했습니다. RSTP에서는 **모든 스위치가 2초(hello time)마다** 직접 BPDU를 만듭니다. hello time은 바꿀 수 있습니다.

### BPDU = Keepalive
Classic은 BPDU를 버리기 전에 **Max Age(20초)** 를 기다렸습니다. RSTP는 OSPF·EIGRP의 hello처럼 BPDU를 **keepalive** 로 씁니다. 이웃 스위치의 BPDU를 **3번 연속 놓치면** 연결이 끊겼다고 보고 MAC 주소를 즉시 지웁니다.

### Inferior BPDU를 받아들인다
Classic은 inferior BPDU를 무시했지만 RSTP는 받아들입니다. Classic Spanning Tree의 **BackboneFast** 와 거의 같은 기능입니다.

### 타이머가 아니라 협상
Classic은 Forwarding 전에 Listening, Learning을 거쳐야 했고 기본 타이머로 **30초** 가 걸렸습니다. RSTP는 타이머 대신 **협상(negotiation)** 으로 Forwarding 여부를 정합니다. 이것이 RSTP에서 가장 중요한 변화, **빠른 수렴(convergence)** 입니다.

### PortFast = Edge port
Classic에서 PortFast를 켜면 Listening·Learning을 건너뛰고 바로 Forwarding이 되며, 포트가 up/down 되어도 Topology Change를 만들지 않습니다. RSTP에서도 PortFast를 그대로 쓰지만 **Edge port** 라고 부릅니다.

### Link type
RSTP가 포트를 빠르게 Forwarding으로 바꿀 수 있는 곳은 **Edge port** 와 **Point-to-point** 링크뿐입니다.

| Link type | Duplex | 설명 |
| --- | --- | --- |
| Point-to-point | Full duplex | 스위치끼리 연결된 보통의 링크 |
| Shared | Half duplex | Hub가 끼어 있는 링크 |

---

## 5. Sync: Proposal / Agreement

![RSTP sync overview](images/rstp-sync-overview.svg)

SW1이 Root Bridge이고 SW2, SW3, SW4는 Non-root 입니다.

1. SW1–SW2 링크가 올라오면 두 포트는 **막힌 상태** 로 시작합니다. SW2가 SW1의 BPDU를 받으면 **Sync** 라는 협상이 시작됩니다.
2. SW2는 Root의 BPDU를 받자마자 **non-edge Designated port를 모두 막습니다.** (Non-edge = 다른 스위치로 가는 포트, Edge = PortFast 포트) 그러면 SW1–SW2 링크가 **Forwarding** 이 됩니다.
3. SW2는 이어서 SW3, SW4와도 Sync를 해서 그쪽 링크도 빠르게 Forwarding으로 만듭니다.

정확히 어떤 메시지가 오가는지 순서대로 보면 이렇습니다.

![Proposal and agreement](images/rstp-proposal-agreement.svg)

| 단계 | 동작 |
| --- | --- |
| 1. Proposal | SW1이 BPDU Flags의 **Proposal 비트** 를 켜서 보냄. SW2는 SW1이 Root라는 걸 알게 됨 |
| 2. Sync | SW2가 **non-edge 포트를 모두 차단** 하고 SW3, SW4 쪽 동기화를 준비 |
| 3. Agreement | SW2가 **Agreement** 를 보냄 (Proposal BPDU 복사본에서 Proposal OFF, Agreement ON). SW2 Fa0/14 Forwarding |
| 4. Forwarding | Agreement를 받은 SW1이 Fa0/14를 **즉시 Forwarding** |
| 5. Proposal | SW2가 Fa0/16(SW3), Fa0/19(SW4)로 같은 Proposal을 보냄 |
| 6. Agreement | SW3, SW4는 다른 포트가 없으므로 바로 Agreement 응답 |
| 7. Forwarding | SW2가 Fa0/16, Fa0/19를 Forwarding으로 바꾸면 끝 |

> 메시지 몇 개가 오가는 것만으로 끝나기 때문에, Classic의 타이머 방식(Listening → Learning → Forwarding)보다 **훨씬 빠릅니다.**

---

## 6. UplinkFast는 기본

Classic에서는 **UplinkFast** 를 직접 켜야 했습니다. RSTP는 기본으로 들어 있습니다. Root port를 잃으면 **Alternate port를 즉시 Forwarding** 으로 바꿉니다.

Classic의 UplinkFast는 모든 스위치의 MAC 주소 테이블을 갱신하려고 multicast 프레임을 보내야 했습니다. RSTP는 Topology Change 방식 자체가 달라서 이게 필요 없습니다.

---

## 7. Topology Change

![Topology change](images/rstp-topology-change.svg)

Classic에서는 **링크 장애도 Topology Change** 였습니다. RSTP에서는 링크 장애를 Topology Change로 보지 않습니다. **non-edge 포트(다른 스위치로 가는 포트)가 Forwarding으로 바뀔 때만** Topology Change 입니다.

Topology Change를 **감지한 스위치** 는:
- 모든 non-edge Designated port와 Root port에 대해 **topology change while 타이머** 를 시작합니다. 값은 **hello time의 2배** 입니다.
- 이 포트들에서 배운 **MAC 주소를 지웁니다.**
- 타이머가 도는 동안, 이 포트들로 보내는 BPDU에 **TC 비트** 를 켭니다. Root port로도 BPDU를 보냅니다.

TC 비트가 켜진 BPDU를 **받은 이웃 스위치** 는:
- BPDU를 받은 포트를 **제외한 모든 포트** 의 MAC 주소를 지웁니다.
- 자기도 topology change while 타이머를 시작하고, 모든 Designated port와 Root port로 TC 비트를 켠 BPDU를 보냅니다.

> Classic처럼 Root Bridge까지 올려 보냈다가 다시 내려오는 게 아니라, **네트워크 전체로 바로 Flooding** 됩니다.

---

## 8. Classic Spanning Tree와의 호환

![Compatibility](images/rstp-compatibility.svg)

RSTP와 Classic Spanning Tree는 **호환됩니다.** 하지만 RSTP 스위치가 Classic 스위치와 맞닿은 구간에서는 **RSTP를 빠르게 만드는 기능들이 동작하지 않습니다.**

위 그림에서 SW1–SW2 사이는 RSTP로 동작하지만, SW2–SW3 사이는 **Classic Spanning Tree로 내려갑니다.**

---

## 핵심 정리

- RSTP(802.1w)는 Classic STP의 **진화**. 설정 방법은 같고 내부 동작이 빨라짐.
- Port State는 **Discarding / Learning / Forwarding** 3개. 새 역할 **Backup port**.
- **Version 2 BPDU**: Flags 전체 사용, Port Role 포함. 모든 스위치가 2초마다 보내고, 3개 놓치면 장애로 판단.
- 타이머 대신 **Proposal → Sync → Agreement** 협상으로 수렴. 빠른 전환은 **Edge port** 와 **Point-to-point** 링크에서만.
- **UplinkFast 기본 내장**, Topology Change는 **non-edge 포트가 Forwarding이 될 때만**, 그리고 **Flooding** 으로 빠르게 전파.
- Classic STP와 호환되지만, 맞닿은 구간은 Classic으로 동작.
