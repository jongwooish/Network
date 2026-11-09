# How to Troubleshoot VLANs and Trunks

**한국어** | [English](README/08-How-to-Troubleshoot-VLANs-and-Trunks.en.md) | [日本語](README/08-How-to-Troubleshoot-VLANs-and-Trunks.ja.md) | [简体中文](README/08-How-to-Troubleshoot-VLANs-and-Trunks.zh-CN.md) | [Español](README/08-How-to-Troubleshoot-VLANs-and-Trunks.es.md) | [Français](README/08-How-to-Troubleshoot-VLANs-and-Trunks.fr.md)

인터페이스 설정(speed / duplex, 에러 등)을 확인했는데도 통신이 안 된다면, 문제는 **VLAN이나 Trunk** 쪽일 수 있습니다. 자주 만나는 5가지 상황을 실습 화면과 함께 정리합니다.

## 점검 순서 한눈에 보기

| 순서 | 확인할 것 | 명령어 |
| --- | --- | --- |
| 1 | 인터페이스가 up / up 인가 | `show ip interface brief`, `show interfaces <if>` |
| 2 | 두 포트가 **같은 VLAN** 인가 | `show vlan` |
| 3 | 포트가 **access / trunk 중 맞는 모드** 인가 | `show interfaces <if> switchport` |
| 4 | Port Security가 막고 있지 않은가 | `show port-security` |
| 5 | **VACL** 이 VLAN 안의 트래픽을 막고 있지 않은가 | `show vlan filter`, `show vlan access-map` |
| 6 | 양쪽 Trunk의 **캡슐화가 같은가** | `show interfaces <if> switchport` |
| 7 | Trunk가 그 VLAN을 **허용**하는가 | `show interfaces <if> trunk` |

---

## 1. VLAN 할당 문제

![스위치 1대 토폴로지](images/troubleshooting/topology-single-switch.png)

VPC11 (192.168.1.1/24) 이 VPC12 (192.168.1.2/24) 로 ping이 안 됩니다. PC 쪽은 문제가 없고, 스위치 문제입니다.

![ping 실패](images/troubleshooting/c1-ping-fail.png)

먼저 인터페이스에 문제가 있는지 봅니다.

![show ip int brief](images/troubleshooting/c1-ip-int-brief.png)

둘 다 up / up 입니다. 이제 VLAN 할당을 확인합니다.

![show vlan](images/troubleshooting/c1-show-vlan.png)

`show vlan` 으로 보면 **Gi0/0은 VLAN 1, Gi0/1은 VLAN 2** 에 있습니다. 같은 서브넷인데 **VLAN이 달라서** 통신이 안 된 겁니다. Gi0/1을 VLAN 1로 옮깁니다.

![VLAN 1로 변경](images/troubleshooting/c1-fix-access-vlan.png)

![ping 성공](images/troubleshooting/c1-ping-ok.png)

> **교훈**: 같은 서브넷인데 통신이 안 되면 `show vlan` 으로 **두 포트의 VLAN이 같은지** 먼저 봅니다.

---

## 2. Switchport 모드 문제

같은 토폴로지, 다른 문제입니다. 인터페이스는 둘 다 up / up 입니다. VLAN 할당을 봅니다.

![show vlan](images/troubleshooting/c2-show-vlan.png)

**Gi0/1은 VLAN 10** 에 있는데, **Gi0/0은 어디에도 보이지 않습니다.** 가능한 원인은 두 가지입니다.
- 인터페이스 자체가 고장 → up / up 이므로 아님
- 인터페이스가 access가 아니라 **trunk** 임 (`show vlan` 은 access 포트만 보여 줌)

![switchport 확인](images/troubleshooting/c2-switchport-trunk.png)

Gi0/0은 **trunk 모드**이고 native VLAN은 1입니다. PC는 802.1Q 태그를 붙이지 않으므로, VPC11의 트래픽은 **native VLAN 1** 로 들어갑니다. 그래서 VLAN 10의 VPC12와 통신이 안 됩니다. access 모드로 바꾸고 VLAN 10에 넣습니다.

![access 모드로 변경](images/troubleshooting/c2-fix-access-mode.png)

![show vlan id 10](images/troubleshooting/c2-show-vlan-id-10.png)

이제 두 포트 모두 VLAN 10에 있습니다. 동작 모드도 확인합니다.

![Operational Mode 확인](images/troubleshooting/c2-operational-mode.png)

![ping 성공](images/troubleshooting/c2-ping-ok.png)

> **교훈**: 포트가 **알맞은 switchport 모드(access / trunk)** 인지 확인합니다. `show vlan` 에 포트가 안 보이면 trunk일 가능성이 큽니다.

---

## 3. VACL (VLAN Access-List) 문제

같은 PC, 같은 스위치, 또 다른 문제입니다. 하나씩 확인해 봅니다.

![show ip int brief](images/troubleshooting/c3-ip-int-brief.png)

인터페이스 up / up. VLAN 할당은?

![show vlan](images/troubleshooting/c3-show-vlan.png)

두 포트 모두 **VLAN 10**. 그럼 Port Security?

![port-security](images/troubleshooting/c3-port-security.png)

Port Security도 없습니다. 인터페이스 정상, 같은 VLAN, Port Security 없음… 그래도 막힌다면 **VACL** 을 의심해야 합니다. VACL은 **같은 VLAN 안의** 트래픽을 허용하거나 차단할 수 있습니다.

![show vlan filter](images/troubleshooting/c3-vlan-filter.png)

VLAN 10에 `BLOCKSTUFF` 라는 VLAN access-map이 걸려 있습니다.

![show vlan access-map](images/troubleshooting/c3-access-map.png)

![show access-lists](images/troubleshooting/c3-access-list.png)

- access-map의 동작(Action)은 **drop** 입니다.
- 매칭 조건은 access-list 1 이고, 그 안의 `permit 192.168.1.0 0.0.0.255` 는 "허용"이 아니라 **"이 범위에 매칭된다"** 는 뜻입니다.
- 두 PC가 이 범위에 있으므로 매칭 → **drop** 됩니다.

동작을 **forward** 로 바꿉니다.

![action forward](images/troubleshooting/c3-fix-action-forward.png)

![ping 성공](images/troubleshooting/c3-ping-ok.png)

> **교훈**: 다른 게 다 정상인데 같은 VLAN끼리 통신이 안 되면 **VACL** 을 확인합니다. VACL 안의 ACL `permit` 은 "매칭"이라는 뜻이고, 실제 허용/차단은 access-map의 **action** 이 정합니다.

---

## 4. Trunk 캡슐화 문제

지금까지는 스위치 1대 안의 문제였습니다. 트래픽이 스위치 여러 대를 지나야 하면 **Trunk** 도 확인해야 합니다.

![스위치 2대 토폴로지](images/troubleshooting/topology-two-switches.png)

양쪽 스위치에 VLAN 10 PC가 하나씩 있는데 서로 통신이 안 됩니다. 먼저 SW1에서 PC 쪽 포트를 봅니다.

![SW1 int g0/0](images/troubleshooting/c4-sw1-int-g0-0.png)

![SW1 g0/0 switchport](images/troubleshooting/c4-sw1-g0-0-switchport.png)

![SW1 port-security](images/troubleshooting/c4-sw1-port-security.png)

up, access, **VLAN 10**, Port Security 없음. SW2도 같은 걸 확인합니다.

![SW2 int g0/0](images/troubleshooting/c4-sw2-int-g0-0.png)

![SW2 g0/0 switchport](images/troubleshooting/c4-sw2-g0-0-switchport.png)

![SW2 port-security](images/troubleshooting/c4-sw2-port-security.png)

PC 쪽 포트는 양쪽 다 정상입니다. 이제 할 수 있는 건 두 가지입니다.
- 같은 스위치에 VLAN 10 PC를 하나 더 연결해서, 스위치 안에서는 통신이 되는지 확인
- **SW1 ↔ SW2 사이 링크** 확인 ← 문제가 생길 곳이 많은 쪽

![SW1 int g0/1](images/troubleshooting/c4-sw1-int-g0-1.png)

![SW2 int g0/1](images/troubleshooting/c4-sw2-int-g0-1.png)

인터페이스는 정상입니다. switchport 정보를 봅니다.

![SW1 g0/1 ISL](images/troubleshooting/c4-sw1-g0-1-isl.png)

![SW2 g0/1 dot1q](images/troubleshooting/c4-sw2-g0-1-dot1q.png)

둘 다 trunk인데, **SW1은 ISL, SW2는 802.1Q** 입니다. 캡슐화가 서로 다르면 상대가 보낸 프레임을 해석하지 못합니다. SW1을 802.1Q로 맞춥니다.

![캡슐화 변경](images/troubleshooting/c4-fix-encapsulation.png)

![ping 성공](images/troubleshooting/c4-ping-ok.png)

> **교훈**: Trunk 양쪽은 **같은 캡슐화**를 써야 합니다.
> 또, 스위치 모델에 따라 기본 모드가 `dynamic auto` 일 수 있는데, **양쪽이 다 `dynamic auto` 면 access** 가 됩니다. Trunk는 `switchport mode trunk` 로 직접 지정하는 게 안전합니다. ([05. DTP](05-Dynamic-Trunking-Protocol.md))

---

## 5. Trunk 허용 VLAN 문제

같은 토폴로지입니다. 다음 항목은 이미 확인했고 문제가 없다고 가정합니다.
- 인터페이스 (speed / duplex)
- Port Security
- Switchport 설정 (VLAN 할당, access 모드)

![ping 실패](images/troubleshooting/c5-ping-fail.png)

스위치 사이 Trunk를 봅니다.

![SW1 g0/1 switchport](images/troubleshooting/c5-sw1-g0-1-switchport.png)

![SW2 g0/1 switchport](images/troubleshooting/c5-sw2-g0-1-switchport.png)

둘 다 trunk, 둘 다 802.1Q. 여기까진 정상입니다. 그런데 Trunk가 동작한다고 해서 **모든 VLAN이 지나갈 수 있는 건 아닙니다.**

![SW1 trunk allowed](images/troubleshooting/c5-sw1-trunk-allowed.png)

![SW2 trunk allowed](images/troubleshooting/c5-sw2-trunk-allowed.png)

`Vlans allowed on trunk` 가 **20** 뿐입니다. VLAN 10은 이 Trunk를 지나갈 수 없습니다. 모든 VLAN을 허용합니다.

![SW1 allowed vlan all](images/troubleshooting/c5-sw1-fix-allowed.png)

![SW2 allowed vlan all](images/troubleshooting/c5-sw2-fix-allowed.png)

![ping 성공](images/troubleshooting/c5-ping-ok.png)

> **교훈**: Trunk가 **모든 VLAN을 허용하는지** 꼭 확인합니다.
> 실무에서는 `all` 대신 필요한 VLAN만 허용하는 경우가 많습니다. 이때 기존 목록에 **추가**하려면 `switchport trunk allowed vlan add 10` 을 씁니다. `add` 없이 `allowed vlan 10` 을 치면 기존 목록이 **덮어쓰기** 되어 다른 VLAN이 끊길 수 있습니다.

---

## 고친 명령어 정리

| 상황 | 원인 | 해결 명령어 |
| --- | --- | --- |
| 1. VLAN 할당 | 두 포트의 VLAN이 다름 | `switchport access vlan 1` |
| 2. Switchport 모드 | PC 포트가 trunk | `switchport mode access` → `switchport access vlan 10` |
| 3. VACL | access-map action이 drop | `vlan access-map BLOCKSTUFF 10` → `action forward` |
| 4. Trunk 캡슐화 | ISL ↔ 802.1Q 불일치 | `switchport trunk encapsulation dot1q` |
| 5. 허용 VLAN | Trunk가 VLAN 20만 허용 | `switchport trunk allowed vlan all` |

## 핵심 정리

- **아래 계층부터** 봅니다: 인터페이스 → VLAN 할당 → switchport 모드 → Port Security → VACL → Trunk.
- `show vlan` 에 포트가 없으면 **trunk** 일 수 있습니다.
- 같은 VLAN인데도 막히면 **VACL** 을 의심합니다.
- Trunk는 **양쪽 모드와 캡슐화**, 그리고 **허용 VLAN 목록**을 함께 확인합니다.
