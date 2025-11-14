# 🚀 11-Router 하이브리드 WAN 통합 프로젝트 (README)

## 1\. 🎯 프로젝트 개요 (Project Overview)

본 프로젝트는 OSPF 멀티 에어리어(Multi-Area)와 EIGRP 자율 시스템(AS)을 통합하고, 중앙 집중식 NAT 및 HSRP 게이트웨이 이중화까지 구현한 **복합 라우팅 기반의 기업 내부망 구축** 시뮬레이션입니다.

M\&A(인수합병) 등으로 인해 서로 다른 라우팅 프로토콜을 사용하는 기업 네트워크를 단일화하고, 고가용성(HA) 및 보안 정책을 적용하는 현업 시나리오를 목표로 합니다.

### 1.1. 📍 최종 아키텍처 (Final Architecture)

본 네트워크는 3개의 독립된 라우팅 도메인과 1개의 인터넷 게이트웨이로 구성됩니다.

| 영역 | 장비 | 프로토콜 | 역할 |
| :---: | :---: | :---: | :--- |
| **백본 (Core)** | R0, R1, R2, R3, R4 | **OSPF Area 0** | 네트워크의 고속 백본 (Core) |
| **인터넷 게이트웨이** | R4 | **NAT / OSPF** | 중앙 집중식 인터넷 접속 및 기본 경로 전파 |
| **지사 그룹 1** | R3, R5, R7, R9 | **EIGRP 100** | OSPF와 재분배되는 지사 그룹 (ASBR) |
| **지사 그룹 2** | R2, L3\_Sw8, L3\_Sw6 | **OSPF Area 1** | HSRP 이중화 지사 (ABR) |

### 1.2. 🚀 핵심 구현 기술

  * **라우팅 재분배 (Redistribution):** `R3`(ASBR)에서 OSPF $\leftrightarrow$ EIGRP 양방향 재분배
  * **OSPF 멀티 에어리어:** `R2`(ABR)를 통한 Area 0 (백본)과 Area 1 (지사) 분리
  * **고가용성 (HSRP):** `L3_Switch_8/6`에서 SVI를 이용한 VLAN별 HSRP Load Balancing
  * **NAT (PAT):** `R4`에서 ROAS를 이용한 중앙 집중식 인터넷 게이트웨이
  * **L2 보안:** `Switch12` 및 하위 스위치에 Native VLAN(999)을 적용하여 L2 충돌 해결
  * **보안 (ACL):** `VLAN 81 (Guest)`이 인터넷(`8.8.8.8`)만 접속하도록 ACL 정책 적용

-----

## 아키텍처 다이어그램
![시스템 다이어그램](images/diagram.png)

-----

## 2\. 🐛 주요 트러블슈팅 및 해결 과정 (Lessons Learned)


### 2.1. 문제 1: EIGRP $\leftrightarrow$ OSPF 간 인터넷 경로 전파 실패

  * **증상:** EIGRP 지사(`R5`)에서 `ping 8.8.8.8` (인터넷) 시 `Destination host unreachable`.
  * **원인:** `R4` (NAT GW)가 광고한 기본 경로(`O*E2 0.0.0.0/0`)를 `R3` (ASBR)가 OSPF로 학습했지만, EIGRP로 재분배하지 않았습니다. EIGRP는 기본적으로 기본 경로를 재분배하지 않습니다.
  * **해결:** `R3`의 EIGRP 인터페이스(`Gi0/2`)에 `ip summary-address eigrp 100 0.0.0.0 0.0.0.0` 명령어를 추가하여 EIGRP 영역으로 기본 경로를 강제 주입했습니다.

### 2.2. 문제 2: HSRP "스플릿 브레인" (Split-Brain)

  * **증상:** `R8`과 `R6`의 `show standby brief` 출력에서 `Standby unknown`이 표시되고, 두 라우터가 모두 `Active` 상태를 주장했습니다.
  * **원인 (Packet Tracer 한계):** `1941` 라우터는 `vlan` 데이터베이스 명령어를 지원하지 않아 `encapsulation dot1Q` (ROAS)가 정상 작동하지 않았습니다. 이로 인해 `Switch12`와 `Native VLAN Mismatch`가 발생하여 HSRP Hello 패킷이 폐기(Drop)되었습니다.
  * **해결:** HSRP 게이트웨이 장비를 `1941 라우터`에서 \*\*`Layer 3 스위치 (3560/3650)`\*\*로 교체했습니다. \*\*SVI (`interface Vlan80`)\*\*를 사용하여 HSRP를 구성함으로써 ROAS의 한계를 극복하고 즉시 문제를 해결했습니다.

### 2.3. 문제 3: OSPF Area 1 $\leftrightarrow$ EIGRP 통신 실패

  * **증상:** `R8` (OSPF Area 1) PC에서 `R5` (EIGRP) PC로 핑 실패 (`Request timed out`).
  * **원인:** OSPF Area 1을 `stub` 영역으로 설정했기 때문입니다. OSPF Stub Area는 외부 경로(`O E2`)를 차단하므로, `R3` (ASBR)가 재분배한 EIGRP 경로(`O E2 192.168.5.0` 등)가 `R2` (ABR)에서 차단되었습니다.
  * **해결:** OSPF Area 1을 `stub`에서 \*\*"Normal Area" (표준 영역)\*\*로 변경(`no area 1 stub`)하여, EIGRP 외부 경로가 Area 1까지 학습되도록 허용했습니다.

-----

## 3\. 💾 최종 설정 스크립트 (Final Configuration)

### 3.1. R4 (Internet GW / NAT)

```cisco
hostname Router4
enable secret class
no ip domain-lookup
ip routing
!
vlan 4
 name R4_LAN
vlan 200
 name ISP_WAN_Link
!
interface GigabitEthernet0/0
 ip address 10.0.1.4 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.2.4 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/2
 no ip address
 no shutdown
!
interface GigabitEthernet0/2.4
 encapsulation dot1Q 4
 ip address 192.168.4.1 255.255.255.0
 ip nat inside
 no shutdown
!
interface GigabitEthernet0/2.200
 encapsulation dot1Q 200
 ip address 200.0.0.2 255.255.255.252
 ip nat outside
 no shutdown
!
interface FastEthernet0/1/0
 switchport mode access
 switchport access vlan 200
 no shutdown
!
interface FastEthernet0/1/1
 switchport mode access
 switchport access vlan 4
 no shutdown
!
interface FastEthernet0/1/3
 switchport mode trunk
 no shutdown
!
router ospf 1
 network 192.168.4.0 0.0.0.255 area 0
 default-information originate
!
ip route 0.0.0.0 0.0.0.0 200.0.0.1
!
ip access-list extended ACL_FOR_NAT
 permit ip 192.168.0.0 0.0.255.255 any
 permit ip 10.3.0.0 0.0.0.255 any
!
ip nat inside source list ACL_FOR_NAT interface GigabitEthernet0/2.200 overload
!
end
```

### 3.2. R3 (ASBR: OSPF $\leftrightarrow$ EIGRP)

```cisco
hostname Router3
enable secret class
no ip domain-lookup
ip routing
!
interface GigabitEthernet0/0
 ip address 10.0.0.3 255.255.255.0
 ip ospf cost 1
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.4.3 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/2
 ip address 10.3.0.3 255.255.255.0
 no shutdown
!
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 10.0.4.0 0.0.0.255 area 0
 redistribute eigrp 100 subnets
!
router eigrp 100
 no auto-summary
 network 10.3.0.0 0.0.0.255
 network 192.168.3.0 0.0.0.255
 redistribute ospf 1 metric 10000 100 255 1 1500
 interface GigabitEthernet0/2
  ip summary-address eigrp 100 0.0.0.0 0.0.0.0
!
end
```

### 3.3. R2 (ABR: OSPF Area 0 $\leftrightarrow$ Area 1)

```cisco
hostname Router2
enable secret class
no ip domain-lookup
ip routing
!
interface GigabitEthernet0/0
 ip address 10.0.3.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.4.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet0/2
 ip address 10.2.0.2 255.255.255.0
 ip ospf 1 area 1
 no shutdown
!
router ospf 1
 network 10.0.3.0 0.0.0.255 area 0
 network 10.0.4.0 0.0.0.255 area 0
 network 10.2.0.0 0.0.0.255 area 1
!
end
```

### 3.4. L3\_Switch\_8 (HSRP Active - VLAN 80)

```cisco
hostname L3_Switch_8
enable secret class
no ip domain-lookup
ip routing
!
vlan 80
 name Staff
vlan 81
 name Guest
vlan 999
 name Native_Dummy
!
interface GigabitEthernet0/1
 no switchport
 ip address 10.2.0.8 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/2
 switchport mode trunk
 switchport trunk native vlan 999
 no shutdown
!
interface Vlan80
 ip address 192.168.8.2 255.255.255.0
 standby 80 ip 192.168.8.1
 standby 80 priority 150
 standby 80 preempt
!
interface Vlan81
 ip address 192.168.81.2 255.255.255.0
 ip access-group GUEST_POLICY_IN in
 standby 81 ip 192.168.81.1
 standby 81 priority 100
 standby 81 preempt
!
router ospf 1
 network 10.2.0.0 0.0.0.255 area 1
 network 192.168.8.0 0.0.0.255 area 1
 network 192.168.81.0 0.0.0.255 area 1
!
ip access-list extended GUEST_POLICY_IN
 permit ip 192.168.81.0 0.0.0.255 host 8.8.8.8
 deny   ip 192.168.81.0 0.0.0.255 192.168.0.0 0.0.255.255
 deny   ip 192.168.81.0 0.0.0.255 10.0.0.0 0.255.255.255
!
end
```

### 3.5. L3\_Switch\_6 (HSRP Active - VLAN 81)

```cisco
hostname L3_Switch_6
enable secret class
no ip domain-lookup
ip routing
!
vlan 80
 name Staff
vlan 81
 name Guest
vlan 999
 name Native_Dummy
!
interface GigabitEthernet0/1
 no switchport
 ip address 10.2.0.6 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/2
 switchport mode trunk
 switchport trunk native vlan 999
 no shutdown
!
interface Vlan80
 ip address 192.168.8.3 255.255.255.0
 standby 80 ip 192.168.8.1
 standby 80 priority 100
 standby 80 preempt
!
interface Vlan81
 ip address 192.168.81.3 255.255.255.0
 ip access-group GUEST_POLICY_IN in
 standby 81 ip 192.168.81.1
 standby 81 priority 150
 standby 81 preempt
!
router ospf 1
 network 10.2.0.0 0.0.0.255 area 1
 network 192.168.8.0 0.0.0.255 area 1
 network 192.168.81.0 0.0.0.255 area 1
!
ip access-list extended GUEST_POLICY_IN
 permit ip 192.168.81.0 0.0.0.255 host 8.8.8.8
 deny   ip 192.168.81.0 0.0.0.255 192.168.0.0 0.0.255.255
 deny   ip 192.168.81.0 0.0.0.255 10.0.0.0 0.255.255.255
!
end
```

### 3.6. Switch12 (Distribution Trunk)

```cisco
hostname Switch12
enable secret class
!
vlan 80
 name Staff
vlan 81
 name Guest
vlan 999
 name Native_Dummy
!
spanning-tree vlan 80,81 priority 0
!
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 999
!
interface GigabitEthernet0/2
 switchport mode trunk
 switchport trunk native vlan 999
!
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 999
!
interface FastEthernet0/2
 switchport mode trunk
 switchport trunk native vlan 999
!
end
```