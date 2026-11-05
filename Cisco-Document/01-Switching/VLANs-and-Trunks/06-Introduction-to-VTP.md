# Introduction to VLAN Trunking Protocol (VTP)

**한국어** | [English](README/06-Introduction-to-VTP.en.md) | [日本語](README/06-Introduction-to-VTP.ja.md) | [简体中文](README/06-Introduction-to-VTP.zh-CN.md) | [Español](README/06-Introduction-to-VTP.es.md) | [Français](README/06-Introduction-to-VTP.fr.md)

스위치가 20대이고 VLAN이 50개라면, 모든 스위치에 VLAN을 하나하나 만드는 건 시간이 너무 오래 걸립니다. **VTP (VLAN Trunking Protocol)** 를 쓰면 한 스위치에서 VLAN을 만들고, 나머지 스위치는 그 정보를 받아 **자동으로 동기화**합니다.

- Cisco 전용 프로토콜
- VTP 정보는 **Trunk 링크**로만 전달됩니다
- 같은 **VTP 도메인 이름**을 가진 스위치끼리만 동기화합니다

---

## 1. VTP 동작 방식

![VTP domain](images/vtp-domain.svg)

1. **VTP Server** 에서 VLAN을 추가 / 수정 / 삭제합니다.
2. 바뀔 때마다 **Configuration Revision** 번호가 1씩 올라갑니다.
3. 최신 정보(advertisement)가 모든 VTP Client로 전달됩니다.
4. VTP Client는 **자기보다 revision이 높은** 정보를 받으면 그 내용으로 동기화합니다.

> VTP Server도 동기화를 합니다(Client 역할도 함께 함). 그래서 VTP Server를 여러 대 두고 아무 Server에서나 VLAN을 바꿀 수 있습니다.

---

## 2. VTP 모드

### VTP Transparent

![VTP transparent](images/vtp-transparent.svg)

VTP Transparent는 advertisement를 **다른 스위치로 전달만** 하고 **자기는 동기화하지 않습니다.** 대신 VLAN을 **자기 스위치에서만** 만들 수 있습니다.

VTP Server에서 VLAN 20을 만들면:
1. VTP Server에서 VLAN 20 생성
2. revision 번호 증가
3. VTP Server가 advertisement를 Transparent 스위치로 보냄
4. Transparent는 동기화하지 않고 advertisement를 Client로 **전달**
5. Client가 최신 정보로 동기화

### 모드 비교

| | **VTP Server** | **VTP Client** | **VTP Transparent** |
| --- | --- | --- | --- |
| VLAN 생성 / 수정 / 삭제 | ✅ | ❌ | 자기 스위치에서만 |
| 스스로 동기화 | ✅ | ✅ | ❌ |
| advertisement 전달 | ✅ | ✅ | ✅ |
| VLAN 정보 저장 위치 | `vlan.dat` | `vlan.dat` | **running-config** (+ `vlan.dat`) |

> 이 밖에 VTP를 아예 끄는 **`off` 모드**도 있습니다. Transparent와 비슷하지만 advertisement를 **전달하지도 않습니다** (VTP 버전 3, 또는 이를 지원하는 IOS).

---

## 3. VTP의 위험: revision 번호

VTP는 편리하지만 큰 위험이 있습니다. **VTP Server도 Client처럼 동기화**하고, 모든 스위치는 **revision 번호가 가장 높은 정보**를 따라갑니다.

이런 일이 생길 수 있습니다.

1. 운영 중인 네트워크에서 VTP Client 스위치 하나를 빼서 실습실로 가져감
2. VTP Server 모드로 바꿈
3. VLAN을 만들고 고치며 이것저것 테스트 → 바꿀 때마다 **revision 증가**
4. 테스트가 끝나 **VLAN을 전부 삭제**
5. 다시 VTP Client 모드로 바꿈
6. 그 스위치를 **운영 네트워크에 다시 연결**

이 스위치의 revision이 운영 네트워크보다 **높기 때문에**, 다른 스위치들이 이 정보로 동기화하고 → **운영 중인 VLAN이 모두 사라집니다.** VLAN에 속해 있던 포트들은 VLAN 1로 돌아가지도 않고, 존재하지 않는 VLAN에 남아 **통신이 끊깁니다.**

> **예방법**
> - 다른 곳에서 쓰던 스위치를 연결하기 전에 **revision을 0으로 초기화**: VTP 도메인 이름을 잠깐 다른 이름으로 바꿨다가 되돌리거나, Transparent 모드로 바꿨다가 되돌립니다.
> - `vtp password` 로 도메인에 비밀번호를 설정합니다.
> - **VTP 버전 3** 을 사용하면 **Primary Server** 로 지정된 스위치만 VLAN 정보를 바꿀 수 있어서 이런 사고를 막을 수 있습니다.
> - VTP가 필요 없으면 **Transparent** (또는 `off`) 로 운영하는 곳도 많습니다.

---

## 4. VTP Pruning

![VTP pruning](images/vtp-pruning.svg)

VLAN 10, 20, 30에 PC가 있고, 스위치 사이는 모든 VLAN을 전달하는 **802.1Q Trunk** 입니다.
VLAN 10의 PC가 **브로드캐스트**를 보내면, 스위치는 flooding하고 Trunk는 모든 VLAN을 전달하므로 **모든 스위치로 퍼집니다.**

그런데 가운데 스위치에는 VLAN 10 PC가 **하나도 없습니다** (VLAN 20만 있음). 이 브로드캐스트는 **대역폭 낭비**입니다.

**VTP Pruning** 을 켜면, 해당 VLAN을 쓰는 장비가 없는 쪽 Trunk로는 그 VLAN 트래픽을 보내지 않습니다.

```
Switch(config)#vtp pruning
```

- Server에서 켜면 도메인 전체에 적용됩니다.
- 스위치 모델에 따라 기본값이 On 또는 Off입니다.
- VLAN 1과 1002 ~ 1005는 prune되지 않습니다.

---

## 5. VTP 설정 실습

![VTP lab topology](images/vtp-lab-topology.svg)

스위치 3대(SW1, SW2, SW3)를 Trunk로 연결했고, 모든 스위치의 VLAN 데이터베이스와 startup-config는 지운 상태에서 시작합니다.

### 5-1. 기본 상태 확인

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

SW2, SW3도 똑같습니다.

| 항목 | 의미 |
| --- | --- |
| `Configuration Revision : 0` | VLAN을 추가·삭제할 때마다 올라감. 아직 아무것도 안 바꿔서 0 |
| `VTP Operating Mode : Server` | 기본 모드는 **Server** |
| `VTP Pruning Mode : Disabled` | Pruning 꺼짐 |
| `VTP version running : 1` | 버전 1~3을 지원하지만 지금은 버전 1로 동작 |
| `VTP Domain Name :` (빈칸) | 도메인 이름이 아직 없음 |

### 5-2. SW1에서 VLAN 만들기

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

SW1의 revision이 **1** 로 올라갔습니다. 하지만 SW2, SW3는 아직 **0** 이고 VLAN 10도 없습니다.
→ **VTP 도메인 이름**이 설정되지 않았기 때문입니다.

### 5-3. VTP 도메인 설정

실시간으로 보기 위해 SW2, SW3에서 debug를 켭니다.

```
SW2#debug sw-vlan vtp events
vtp events debugging is on
```

SW1에 도메인 이름을 설정합니다.

```
SW1(config)#vtp domain JJONG
Changing VTP domain name from NULL to JJONG
```

SW3의 debug 출력:

```
VTP LOG RUNTIME: Summary packet received in NULL domain state
VTP LOG RUNTIME: Summary packet received, domain = JJONG, rev = 1, followers = 1, length 77, trunk Gi0/0
VTP LOG RUNTIME: Transitioning from NULL to JJONG domain
VTP LOG RUNTIME: Summary packet rev 1 greater than domain JJONG rev 0
```

- 도메인 이름이 **없던(NULL)** 스위치는 받은 VTP 패킷의 도메인 이름(`JJONG`)을 **자기 도메인으로 바꿉니다.** (이미 도메인 이름이 있으면 바꾸지 않습니다.)
- 받은 패킷의 revision(**1**)이 자기 것(**0**)보다 높으므로 **동기화합니다.**

debug는 출력이 많으니 꼭 끕니다.

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

SW2, SW3가 VTP로 **VLAN 10을 배웠습니다.**

### 5-4. 여러 Server에서 VLAN 만들기

지금은 세 스위치 모두 Server라서 아무 스위치에서나 VLAN을 만들 수 있습니다.

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

SW2, SW3도 똑같이 10, 20, 30을 가지고, revision은 세 스위치 모두 **3** 입니다. VLAN을 만들 때마다 1씩 올라갑니다.

### 5-5. VTP Client 모드

```
SW2(config)#vtp mode client
Setting device to VTP Client mode for VLANS.
```

```
SW2#show vtp status | include Operating
VTP Operating Mode                : Client
```

이제 SW1·SW3는 Server, SW2는 Client입니다. 테스트를 위해 **SW1 ↔ SW3 링크를 끊어서** 둘 사이에 직접 연결이 없게 합니다. (SW1 — SW2 — SW3)

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

SW3도 VLAN 40을 배웁니다. SW1 → **SW2(Client)** → SW3 경로로 왔습니다.
→ **Client도 스스로 동기화하고, advertisement를 다른 스위치로 전달합니다.**

Client에서는 VLAN을 만들 수 없습니다.

```
SW2(config)#vlan 50
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

### 5-6. VTP Transparent 모드

SW1 ↔ SW3 링크는 계속 끊어 둔 상태에서 SW2를 Transparent로 바꿉니다.

```
SW2(config)#vtp mode transparent
Setting device to VTP Transparent mode for VLANS.
```

SW1에서 VLAN 50을 만듭니다.

```
SW1(config)#vlan 50
SW1(config-vlan)#name Research
```

| 스위치 | VLAN 50 | 이유 |
| --- | --- | --- |
| SW1 (Server) | ✅ 있음 | 직접 만듦 |
| SW2 (Transparent) | ❌ 없음 | Transparent는 **동기화하지 않음** |
| SW3 (Server) | ✅ 있음 | SW2가 advertisement를 **전달**해 줌 |

이번엔 SW2(Transparent)에서 VLAN 60을 만듭니다.

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

Transparent는 **자기 스위치에서는** VLAN을 만들 수 있습니다. 하지만 SW1, SW3에는 VLAN 60이 **나타나지 않습니다.** Transparent의 VLAN은 그 스위치에서만 쓰이고 광고되지 않기 때문입니다.

### 5-7. Transparent는 VLAN을 running-config에 저장

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

| 모드 | VLAN 정보 저장 위치 |
| --- | --- |
| Server / Client | `flash:vlan.dat` (running-config에 안 나옴) |
| **Transparent** | **running-config** 에도 나옴 → `copy run start` 로 저장 |

---

## 6. 명령어 정리

| 명령어 | 설명 |
| --- | --- |
| `vtp domain <이름>` | VTP 도메인 이름 설정 (대소문자 구분) |
| `vtp mode {server \| client \| transparent \| off}` | VTP 모드 변경 |
| `vtp version {1 \| 2 \| 3}` | VTP 버전 변경 |
| `vtp password <비밀번호>` | 도메인 비밀번호 설정 |
| `vtp pruning` | VTP Pruning 켜기 |
| `show vtp status` | 버전, 도메인, 모드, revision 확인 |
| `show vtp password` | 설정된 비밀번호 확인 |
| `debug sw-vlan vtp events` | VTP 이벤트 실시간 확인 (`no debug all` 로 끄기) |

---

## 7. 핵심 정리

- VTP는 한 스위치에서 만든 VLAN을 **같은 도메인의 스위치들에 자동으로 동기화**합니다. **도메인 이름이 같아야** 동작하고, **Trunk** 로만 전달됩니다.
- **Server**: VLAN 생성 가능 + 동기화 + 전달 / **Client**: 동기화 + 전달 / **Transparent**: 전달만, VLAN은 자기 것만
- 모든 스위치는 **revision이 가장 높은** 정보를 따라갑니다. 다른 곳에서 쓰던 스위치는 연결 전에 **revision을 0으로** 만들어야 VLAN 전체 삭제 사고를 막을 수 있습니다.
- **VTP Pruning** 은 필요 없는 Trunk로 가는 VLAN 트래픽을 막아 대역폭을 아낍니다.
- Transparent는 VLAN을 **running-config** 에, Server/Client는 **vlan.dat** 에 저장합니다.
