# VTP Version 3

**한국어** | [English](README/07-VTP-Version-3.en.md) | [日本語](README/07-VTP-Version-3.ja.md) | [简体中文](README/07-VTP-Version-3.zh-CN.md) | [Español](README/07-VTP-Version-3.es.md) | [Français](README/07-VTP-Version-3.fr.md)

VTP 버전 3의 목적도 VLAN 동기화로 같지만, 버전 1·2의 약점을 고치고 여러 기능이 추가되었습니다. 예전 Catalyst IOS에서는 지원되지 않다가 최근 IOS부터 쓸 수 있습니다.

> VTP 기본 개념(Server / Client / Transparent, revision)은 [06. Introduction to VTP](06-Introduction-to-VTP.md) 를 먼저 보세요.

## VTP 버전 3에서 달라진 점

| 기능 | 설명 |
| --- | --- |
| **Primary Server** | Primary Server만 VLAN을 생성 / 수정 / 삭제할 수 있음 → v1·v2처럼 **실수로 VLAN이 전부 지워지는 사고**를 막음 |
| **Extended VLAN** | 확장 범위 VLAN **1006 ~ 4094** 도 동기화 |
| **Private VLAN** | Private VLAN 정보 동기화 |
| **RSPAN VLAN** | Remote SPAN VLAN 동기화 |
| **MST 지원** | 스위치마다 따로 하던 **MST 설정**을 동기화 |
| **인증 강화** | 비밀번호를 **hidden / secret** 으로 저장 가능 |
| **VTP mode off** | VTP를 전역 또는 인터페이스 단위로 **완전히 끔** |
| **호환성** | 버전 **2와는 호환**, 버전 1과는 호환되지 않음 |

---

## 실습 토폴로지

![VTP version 3 lab topology](images/vtp3-topology.svg)

SW1, SW2, SW3 사이 링크는 모두 Trunk입니다.

---

## 1. 기본 설정

### 1-1. 도메인 이름이 먼저

```
SW1(config)#vtp version 3
```

VTP 버전 3은 **도메인 이름이 비어 있으면(NULL) 켜지지 않습니다.** 도메인부터 설정합니다.

```
SW1(config)#vtp domain JJONG
SW1(config)#vtp version 3
```

SW2, SW3도 똑같이 설정합니다. 세 스위치 모두 기본값인 **VTP Server** 모드로 동작합니다.

### 1-2. Server만으로는 VLAN을 못 만든다

```
SW1(config)#vlan 10
```

VTP 버전 3에서는 Server라도 **Primary Server가 아니면** VLAN 데이터베이스를 바꿀 수 없어서, 이 명령은 거부됩니다.

### 1-3. Primary Server 지정

```
SW1#vtp primary vlan
```

- **설정 모드가 아니라 privileged 모드(`#`)** 에서 실행합니다.
- 확인(confirm)을 누르면 SW1이 **VLAN 기능의 Primary Server** 가 되고, 다른 스위치에도 Primary Server가 바뀌었다는 로그가 뜹니다.

```
SW1#show vtp status
```

`Feature VLAN` 항목에서 `VTP Operating Mode : Primary Server` 와 Primary ID(MAC 주소)를 확인할 수 있습니다. SW2, SW3에서 봐도 **SW1이 Primary Server** 로 표시됩니다.

### 1-4. 도메인의 모든 스위치 보기

```
SW1#show vtp devices
```

VTP 버전 3에 새로 생긴 명령입니다. CDP처럼 직접 연결된 장비만이 아니라 **같은 VTP 도메인의 모든 스위치**를 보여 줍니다.

### 1-5. VLAN 동기화

```
SW1(config)#vlan 100
```

SW2, SW3에서 `show vlan brief` 로 보면 **VLAN 100** 이 생겼습니다.

### 1-6. 확장 범위 VLAN (1006 ~ 4094)

```
SW1(config)#vlan 2000
```

```
SW2#show vlan brief | include 2000
```

버전 1·2에서는 확장 VLAN을 VTP로 보낼 수 없었지만, 버전 3에서는 **2000번도 그대로 동기화**됩니다.

---

## 2. Private VLAN

VTP 버전 3은 Private VLAN 정보를 동기화합니다. 단, **VLAN 데이터베이스 정보만** 보내고 포트 설정은 보내지 않습니다.

```
SW1(config)#vlan 501
SW1(config-vlan)#private-vlan community
SW1(config-vlan)#vlan 502
SW1(config-vlan)#private-vlan isolated
SW1(config-vlan)#vlan 500
SW1(config-vlan)#private-vlan primary
SW1(config-vlan)#private-vlan association 501,502
```

| VLAN | 종류 |
| --- | --- |
| 500 | Primary VLAN |
| 501 | Community VLAN |
| 502 | Isolated VLAN |

```
SW2#show vlan private-vlan
```

SW1에서 만든 Private VLAN 구성이 SW2, SW3에도 **똑같이** 나타납니다.

---

## 3. Remote SPAN (RSPAN)

RSPAN VLAN도 특별한 종류의 VLAN인데, 이제 VTP로 동기화됩니다.

```
SW1(config)#vlan 600
SW1(config-vlan)#remote-span
```

```
SW3#show vlan remote-span
```

모든 스위치에서 VLAN 600이 **RSPAN VLAN** 으로 보입니다.

---

## 4. MST (Multiple Spanning-Tree)

예전에는 MST 설정을 스위치마다 똑같이 손으로 넣어야 했습니다. VTP 버전 3은 MST를 별도의 **feature** 로 동기화합니다.

| Feature | 동기화하는 것 | 기본 모드 |
| --- | --- | --- |
| **VLAN** | VLAN, 확장 VLAN, Private VLAN, RSPAN | Server |
| **MST** | MST 설정 | Transparent |
| **UNKNOWN** | 앞으로 추가될 기능용 자리 | Transparent |

MST도 **Primary Server** 가 필요합니다. VLAN의 Primary와 같은 스위치여도 되고 다른 스위치여도 됩니다. 여기서는 **SW2** 를 MST Primary로 정합니다.

```
SW2(config)#vtp mode server mst
SW2(config)#exit
SW2#vtp primary mst
```

```
SW1(config)#vtp mode client mst
SW3(config)#vtp mode client mst
```

SW2에서 MST 설정을 만듭니다.

```
SW2(config)#spanning-tree mst configuration
SW2(config-mst)#name JJONG
SW2(config-mst)#revision 1
SW2(config-mst)#instance 1 vlan 10,20
SW2(config-mst)#instance 2 vlan 30,40
```

모든 스위치에서 MST를 켭니다.

```
SW1(config)#spanning-tree mode mst
SW2(config)#spanning-tree mode mst
SW3(config)#spanning-tree mode mst
```

```
SW1#show spanning-tree mst configuration
```

SW1, SW3도 SW2에서 만든 MST 설정을 받았고, **running-config에도 저장**됩니다. 이제 MST 설정을 복사·붙여넣기 할 필요가 없습니다.

---

## 5. UNKNOWN feature

`show vtp status` 에는 VLAN, MST 외에 **UNKNOWN** feature도 있습니다. 나중에 VTP 버전 3이 새 기능을 쓸 때를 위한 자리입니다.

- 지금은 **Transparent 모드만** 가능합니다.
- `vtp mode server unknown` 이나 `vtp mode client unknown` 은 오류가 납니다.

```
SW1(config)#vtp mode transparent unknown
```

---

## 6. 인증 (Authentication)

버전 3에서는 비밀번호를 **hidden** 으로 저장할 수 있습니다.

```
SW1(config)#vtp password <PASSWORD> hidden
```

```
SW1#show vtp password
```

hidden으로 설정하면 비밀번호가 평문이 아니라 **암호화된 16진수 문자열**로만 보이고, 스위치에서 원래 비밀번호를 꺼낼 수 없습니다.
새 스위치를 도메인에 넣을 때는 그 문자열을 **secret** 으로 그대로 넣습니다.

```
SW4(config)#vtp password <암호화된 문자열> secret
```

> Primary Server로 지정할 때(`vtp primary`)도 비밀번호를 물어봅니다. hidden 비밀번호는 Primary 변경까지 막아 주는 장치입니다.

---

## 7. VTP mode off

버전 3은 VTP를 **완전히 끄는** `off` 모드를 지원합니다.

| 모드 | 자기 동기화 | advertisement 전달 |
| --- | --- | --- |
| Transparent | ❌ | ✅ 계속 전달 |
| **Off** | ❌ | ❌ **전달도 안 함** |

전역으로 끄기 (feature별로 지정 가능):

```
SW1(config)#vtp mode off
SW1(config)#vtp mode off vlan
SW1(config)#vtp mode off mst
```

인터페이스 단위로 끄기:

```
SW1(config)#interface GigabitEthernet0/1
SW1(config-if)#no vtp
```

이 인터페이스는 더 이상 VTP에 참여하지 않습니다.

---

## 8. 하위 호환성

- VTP 버전 3은 **버전 2와 호환**되고, **버전 1과는 호환되지 않습니다.**
- 버전 3 스위치가 버전 2 advertisement를 받으면, 버전 2와 호환되는 advertisement를 전달합니다.
- 버전 2는 새 비밀번호 방식을 모르므로, 섞어 쓸 때는 **버전 3 스위치의 비밀번호를 해제**해야 합니다.
- 동작이 이상하면 debug로 확인합니다. (원문 실습에서는 Catalyst 3750의 오래된 IOS에서 VTP 수신이 안 되는 문제가 있었습니다.)

```
SW1#debug sw-vlan vtp events
SW1#no debug all
```

---

## 명령어 정리

| 명령어 | 설명 |
| --- | --- |
| `vtp domain <이름>` | 도메인 이름 설정 (버전 3은 필수) |
| `vtp version 3` | VTP 버전 3으로 변경 |
| `vtp primary [vlan \| mst]` | Primary Server 지정 (privileged 모드) |
| `vtp mode {server \| client \| transparent \| off} [vlan \| mst \| unknown]` | feature별 VTP 모드 |
| `vtp password <비밀번호> [hidden \| secret]` | 비밀번호 설정 |
| `no vtp` (interface) | 해당 인터페이스에서 VTP 끄기 |
| `show vtp status` | 모드, Primary Server, revision 확인 |
| `show vtp devices` | 도메인의 모든 VTP 스위치 보기 |
| `show vtp password` | 비밀번호 확인 (hidden이면 암호화 문자열) |

---

## 핵심 정리

- VTP 버전 3은 **도메인 이름이 필수**이고, **Primary Server만** VLAN을 바꿀 수 있어 VLAN 전체 삭제 사고를 막습니다.
- **확장 VLAN, Private VLAN, RSPAN VLAN, MST 설정**까지 동기화합니다.
- 기능이 **VLAN / MST / UNKNOWN feature** 로 나뉘고, feature마다 모드와 Primary Server를 따로 정합니다.
- **hidden 비밀번호**, **off 모드**(전역·인터페이스)가 추가되었습니다.
- **버전 2와만** 호환됩니다.
