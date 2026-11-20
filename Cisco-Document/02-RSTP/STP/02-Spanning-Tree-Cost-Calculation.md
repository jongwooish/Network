# Spanning-Tree Cost Calculation

**한국어** | [English](README/02-Spanning-Tree-Cost-Calculation.en.md) | [日本語](README/02-Spanning-Tree-Cost-Calculation.ja.md) | [简体中文](README/02-Spanning-Tree-Cost-Calculation.zh-CN.md) | [Español](README/02-Spanning-Tree-Cost-Calculation.es.md) | [Français](README/02-Spanning-Tree-Cost-Calculation.fr.md)

Non-root 스위치는 Root Bridge까지 가는 **가장 짧은 경로**를 찾아야 합니다. Ethernet, FastEthernet, Gigabit처럼 서로 다른 인터페이스가 섞여 있으면 어떻게 될까요? 이 문서에서 직접 계산해 봅니다.

> Root Bridge, Root port 같은 기본 개념은 [01. Introduction to Spanning-Tree](01-Introduction-to-Spanning-Tree.md) 를 먼저 보세요.

---

## 1. Cost와 Root Path Cost

![Topology with mixed link speeds](images/stp-cost-topology.svg)

위 그림은 스위치가 여러 대인 더 큰 네트워크입니다. 인터페이스 종류도 Ethernet(10 Mbit), FastEthernet(100 Mbit), Gigabit(1000 Mbit)으로 다양합니다. 맨 위의 SW1이 Root Bridge이므로, 나머지 스위치는 모두 non-root이고 Root Bridge까지의 가장 짧은 경로를 찾아야 합니다.

Spanning-tree는 **cost**로 Root Bridge까지의 가장 짧은 경로를 정합니다. **인터페이스가 느릴수록 cost가 높고**, cost가 가장 낮은 경로가 사용됩니다.

| Bandwidth | Cost |
| --- | --- |
| 10 Mbit | 100 |
| 100 Mbit | 19 |
| 1000 Mbit | 4 |

Cost 값은 BPDU 안에 있습니다.

![BPDU root path cost field](images/stp-cost-bpdu-root-path-cost.svg)

BPDU에는 **Root Path Cost**라는 필드가 있습니다. 각 스위치는 이 필드에 자신이 Root Bridge까지 가는 가장 짧은 경로의 cost를 넣습니다. 스위치들은 어떤 스위치가 Root Bridge인지 알게 되면 그곳까지 가는 가장 짧은 경로를 찾습니다. **BPDU는 Root Bridge에서 아래쪽으로** 모든 스위치에 전달됩니다.

CCNA나 CCNP를 공부했다면 cost 이야기가 익숙할 수 있습니다. **OSPF**도 cost로 목적지까지의 가장 짧은 경로를 계산합니다. 하지만 큰 차이가 있습니다. OSPF는 **토폴로지 데이터베이스(LSDB)**를 만들어서 모든 라우터가 네트워크 모양을 정확히 압니다. 반면 spanning-tree는 "단순"합니다. **스위치는 토폴로지 모양을 전혀 모릅니다.** Root Bridge에서 내려오는 BPDU만 보고 판단합니다.

---

## 2. 단계별 계산

이 토폴로지에서 cost가 어떻게 계산되는지 단계별로 보겠습니다.

![Step 1](images/stp-cost-step1.svg)

- SW2는 SW1과 직접 연결된 링크를 **Root port**로 씁니다. 100 Mbit 인터페이스이고 cost가 19입니다.
- SW2는 SW4 쪽으로 BPDU를 보내고, BPDU의 Root Path Cost 필드에는 **19**가 들어갑니다.
- SW3도 SW1에게서 BPDU를 받기 때문에, **지금 이 순간에는** 10 Mbit 인터페이스를 Root port로 고를 수 있습니다. 계속 보겠습니다.

![Step 2](images/stp-cost-step2.svg)

이 그림은 설명이 조금 필요합니다.

- SW3는 10 Mbit 인터페이스(cost 100)와 1000 Mbit 인터페이스(cost 4)로 BPDU를 받습니다. **1000 Mbit 인터페이스를 Root port**로 씁니다(Root Bridge까지 가장 짧은 경로는 19 + 19 + 4 = **42**).
- 처음에 SW3가 SW4로 보낸 BPDU의 Root Path Cost는 **100**입니다.
- SW4는 SW2에게서 Root Path Cost 19인 BPDU를, SW3에게서 Root Path Cost 100인 BPDU를 받습니다.
- SW2를 거치는 경로가 더 짧으므로 **SW2 쪽이 SW4의 Root port**가 됩니다.
- SW4는 SW3와 SW5 쪽으로 BPDU를 보냅니다. Root Path Cost는 **38**입니다(자신의 Root Path Cost 19 + 자기 인터페이스 cost 19).
- SW3는 SW5 쪽으로 BPDU를 보내고 Root Path Cost에 **42**(19 + 19 + 4)를 넣습니다.

![Step 3](images/stp-cost-step3.svg)

전체 그림은 위와 같습니다. SW5는 SW3와 SW4에게서 BPDU를 받고, Root Path Cost 필드에서 다음 값을 봅니다.

- SW3의 BPDU: cost 42
- SW4의 BPDU: cost 38

SW5는 여기에 자기 인터페이스 cost를 더합니다. SW4를 거치면 38 + 19(100 Mbit) = **57**, SW3를 거치면 42 + 100(10 Mbit) = **142**입니다. 그래서 **SW4 쪽 인터페이스를 Root port**로 고릅니다.

여기까지 따라오셨나요? 꼭 기억할 점은 **스위치는 받은 BPDU만 보고 판단한다**는 것입니다. 스위치는 토폴로지 모양을 모릅니다. 아는 것은 **어느 인터페이스로 가장 좋은 BPDU를 받았는지**뿐이고, 가장 좋은 BPDU는 Root Bridge까지의 경로가 가장 짧은 BPDU입니다.

---

## 3. Cost가 같으면?

![Equal cost](images/stp-cost-equal-cost.svg)

SW1은 Root Bridge이고 SW2는 non-root입니다. 이중화를 위해 두 스위치 사이에 링크가 두 개 있습니다. 이중화는 루프를 뜻하므로 spanning-tree는 SW2의 인터페이스 하나를 차단합니다.

SW2는 두 인터페이스 모두로 BPDU를 받지만 **Root Path Cost가 같습니다.** 그렇다면 Fa0/1과 Fa0/2 중 어느 쪽을 차단할까요? Cost가 같으면 spanning-tree는 **포트 priority**를 봅니다. 기본으로 모든 인터페이스의 포트 priority가 같으므로 **인터페이스 번호**로 결정됩니다.

**가장 낮은 인터페이스 번호**가 선택되므로 여기서는 Fa0/2가 차단됩니다. 물론 포트 priority는 바꿀 수 있어서, 어느 인터페이스를 차단할지 직접 정할 수도 있습니다.

Spanning-tree는 무언가를 결정해야 할 때 항상 아래 순서를 따릅니다.

1. **가장 낮은 Bridge ID**: Bridge ID가 가장 낮은 스위치가 Root Bridge가 됩니다.
2. **Root Bridge까지 가장 낮은 path cost**: BPDU를 여러 개 받으면, Root Bridge까지 cost가 가장 낮은 인터페이스를 Root port로 고릅니다.
3. **가장 낮은 송신자 Bridge ID**: Root Bridge로 갈 수 있는 스위치 두 대와 연결되어 있고 cost가 같으면, Bridge ID가 더 낮은 스위치와 연결된 인터페이스를 Root port로 고릅니다.
4. **가장 낮은 송신자 Port ID**: 같은 스위치와 인터페이스 두 개로 연결되어 있고 cost가 같으면, 번호가 가장 낮은 인터페이스를 Root port로 고릅니다.

이 목록은 적어두고 꼭 기억해 두세요.

---

## 핵심 정리

- Spanning-tree는 **cost**로 Root Bridge까지의 가장 짧은 경로를 고릅니다. 느린 인터페이스일수록 cost가 높습니다(10 Mbit = 100, 100 Mbit = 19, 1000 Mbit = 4).
- 각 스위치는 BPDU의 **Root Path Cost** 필드에 자신의 경로 cost를 넣어 아래쪽으로 전달합니다.
- 받은 Root Path Cost에 **자기 인터페이스 cost를 더해서** 비교합니다(예: SW5 → SW4 경유 38 + 19 = 57).
- OSPF와 달리 스위치는 **토폴로지를 모르고**, 받은 BPDU만 보고 판단합니다.
- 결정 순서: **Bridge ID → Root path cost → 송신자 Bridge ID → 송신자 Port ID**, 모두 낮을수록 좋습니다.
