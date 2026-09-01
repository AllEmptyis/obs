# 참고
- AEU는 default gw를 지정하지 않으면 서비스를 타지 않음
	- routing도 하지 않음
- vrrp 서비스는 mp로 처리한다 (포트바운더리 필요x)
- virtual bridge 구성 설명
```
vlan은 인터페이스마다 설정 (3개)
ip 대역 하나만 사용
ip 대역 하나만 사용하기 위해 fw과 연결되는 인터페이스를 /32로 설정한다

원래는 각 인터페이스마다 서로 다른 네트워크 대역을 부여해야 하지만
virtual bridge 구성은 인터페이스는 여러 개 만들지만 /32로 설정해서
하나의 대역인것처럼 설정할 수 있음
```

- fwlb에서 pask가 트래픽이 서버에서 왔는지 fw에서 왔는지 아는 방법
	- pask fwlb에 설정되어 있는 필터의 dip,sip를 참고한다
	- 인입된 패킷의 dip가 필터의 sip인 경우 리버스
	- 인입된 패킷의 sip가 필터의 sip에 매칭되는 경우 포워드

- 주의
	- 이중화 구성에서 클라이언트 gw vrrp vip로 설정 주의

- 인터페이스를 나눠야 하는 이유
	- fwlb는 인터페이스를 보고 포워딩 해주기 때문
	- real을 보내는 기준이 인터페이스

- FWLB에서 health check 하는 방법
	- H/C도 fwlb 서비스 방식으로 처리함 (라우팅 처리x)
```
h/c tip로 상단>하단 검사할 때 real 인터페이스를 대상으로 두 가지 경로로 h/c를 진행
어떻게 경로 유지를 하고 보낼 수 있는지?
-> 헬스체크 패킷에 인터페이스를 알 수 있는 필드가 존재
```
# 구성
![[image-6.png|659]]

- 방화벽과 연결되는 구간- untag
- 여러 vlan이 지나가야 되는 구간을 tag로 설정해주면 된다
- 현재 구성의 경우 ge1을 통해 서비스

# 동작 요약
- 

# 설정
## pas-k (ext)
- host routing, proxy arp 설정 필요

- 포트 바운더리
	- 서비스 트래픽이 흐르는 구간은 전부 설정
```
ext_2# show port-boundary

================================================================================
PORT-BOUNDARY Configuration
 ------------------------------------------------------------------------------
  ID    Status Type    Boundary Promisc Include MAC Port List Description
  1     enable include all      off     none        ge1,ge11
================================================================================

ext_2#
ext_2#
ext_1(config)# exit
ext_1# show port-boundary

================================================================================
PORT-BOUNDARY Configuration
 ------------------------------------------------------------------------------
  ID    Status Type    Boundary Promisc Include MAC Port List    Description
  1     enable include all      off     none        ge1,ge9,ge11
================================================================================
```

- 라우팅
	- gw가 지정되어 있어야 서비스 수행
```
ext_1# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 1.1.1.254

    Network
       Destination Gateway Interface Priority HC-ID HC-Type HC-Result Description
       1.1.1.11/32 0.0.0.0 fw
       1.1.1.12/32 0.0.0.0 fw2
       1.1.1.0/24  0.0.0.0 ext
================================================================================
```

- vlan
```
ext_1# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                                      | xg  |          |         |
          |         |                   1 1 1 1 1 1 1 1 1 1 2 |     |          |         |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 | 1 2 | TBM      | ND      | Description
--------------------------------------------------------------------------------
default   | 1       | . u u u u u u u . u . u u u u u u u u u | u u | disable  | 1       |
ext       | 10      | t . . . . . . . u . . . . . . . . . . . | . . | disable  | 1       |
fw        | 20      | t . . . . . . . . . u . . . . . . . . . | . . | disable  | 1       |
fw2       | 30      | t . . . . . . . . . . . . . . . . . . . | . . | disable  | 1       |
================================================================================

ext_2# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                                      | xg  |          |
          |         |                   1 1 1 1 1 1 1 1 1 1 2 |     |          |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 | 1 2 | TBM      | Description
--------------------------------------------------------------------------------
default   | 1       | . u u u u u u u u u . u u u u u u u u u | u u | disable  |
ext       | 10      | t . . . . . . . . . . . . . . . . . . . | . . | disable  |
fw        | 20      | t . . . . . . . . . . . . . . . . . . . | . . | disable  |
fw2       | 30      | t . . . . . . . . . u . . . . . . . . . | . . | disable  |
================================================================================
```

- 인터페이스
```
ext_1# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address     Broadcast       RPF     ND    Description
  fw      up     00:06:c4:94:1c:0f 1.1.1.1/32       1.1.1.1         default 1
  ext     up     00:06:c4:94:1c:0f 1.1.1.1/24       1.1.1.255       default 1
  fw2     up     00:06:c4:94:1c:0f 1.1.1.1/32       1.1.1.1         default 1
  default down   00:06:c4:94:1c:0f                                  default 1
  mgmt    up     00:06:c4:94:1c:0e 192.168.100.1/24 192.168.100.255 default 0
================================================================================


ext_2# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address     Broadcast       RPF     Description
  fw      up     00:06:c4:94:1c:03 1.1.1.2/32       1.1.1.2         default
  ext     up     00:06:c4:94:1c:03 1.1.1.2/24       1.1.1.255       default
  fw2     up     00:06:c4:94:1c:03 1.1.1.2/32       1.1.1.2         default
  default down   00:06:c4:94:1c:03                                  default
  mgmt    up     00:06:c4:94:1c:02 192.168.100.1/24 192.168.100.255 default
================================================================================
```

- vrrp
	- vrrp는 3개의 인터페이스에 대해 모두 같은 vip(1.1.1.250) 설정
		- 왜냐하면 어차피 장비가 하나의 대역(1.1.1.x)만 가지고 있기 때문
	- vrrp vip로 H/C 설정함 (이중화인 경우), 혹은 환경에 따라 gw 등으로 설정하기도 함
	- vrrp가 advertisement 보낼 때는 자신의 인터페이스 ip로 전송, failover 되었을 때 garp 보낼 때만 vrrp vip로 전송한다
```
failover
  delay-time 10
  session-sync status disable
  session-sync interval 100
  session-sync full-interval 30
  session-sync update live
  session-sync peer node2
  session-sync interface hc-retry 3
  active-active-failover method disable
  apply
  vrrp 1
    ndomain 1
    mode active-standby
    status enable
    priority 100
    send-garp-all-svip disable
    interface fw vip 1.1.1.250
    interface fw advertise-send enable
    interface ext vip 1.1.1.250
    interface ext advertise-send enable
    interface fw2 vip 1.1.1.250
    interface fw2 advertise-send enable
    advertise-interval 10
    retry 3
    arp-count 0
    vmac enable
    preemption enable
    track single-port 1 port ge11
    apply
  apply
  ha
    status disable
    default-state master
    heartbeat-interval 10
    retry 3
    vmac enable
    apply
  apply
  ssl-session-cache-sync
    status disable
    apply
  forwarding-standby disable
  apply
  exit
```

- real
	- real 설정 시 중요한 것은 실제 방화벽이 연결되어 있는 인터페이스를 지정하는 것
	- fwlb 수행할 때 연결된 인터페이스(fw)로 포워딩
```
ext_1(config)# show real

================================================================================
REAL Configuration
 ------------------------------------------------------------------------------
  ID    Name  RIP      Rport SSL-Rport Weight Backup SVC-IP Status Description
  1     fw1   1.1.1.11                 1                    enable
  2     fw2   1.1.1.12                 1                    enable
================================================================================
```

- health check
	- tip: target ip
		- internal의 vrrp vip로 설정
```
ext_1(config)# show health-check 1

================================================================================
  HEALTH-CHECK: 1
 ------------------------------------------------------------------------------
    ID                   : 1
    Type                 : icmp
    Timeout              : 3
    Interval             : 5
    Retry                : 3
    Recover              : 0
    Status               : enable
    Graceful Shutdown    : disable
    Description          :

    --- Option ---------------------------------------------------------------
    SIP                  :
    TIP                  : 2.2.2.250 //internal vrrp vip
    DIP                  :
    Increase ICMP ID     : disable
================================================================================
```

- FWLB
	- filter 설정 방법 (exclude가 항상 우선 적용)
		- exclude - 방화벽으로 직접 통신할 수 있도록 필터에 매칭 되는 경우 제외
```
ext_1(config)# show info fwlb ext

================================================================================
  FWLB: ext
 ------------------------------------------------------------------------------
    Name                 : ext
    IP Version           : ipv4
    Ndomain              : 1
    Status               : enable
    Priority             : 100
    LB Method            : rr
    VPNLB                : disable
    Multi Tunnel         : disable
    Branch Relay         : disable
    Position             : internal
    Fail Skip            : none
    Session Timeout Mode : global
    Session Reset        : none
    Session-sync         : none
    H/C Condition        : all
    Health Check         : 1
    Passive Health Check :
    Service Health       : ACT
    Description          :

    Filter
       ID    Type    Protocol Src IP     Dst IP      Status Hit   Description
       1     include all      1.1.1.0/24 2.2.2.0/24  enable 38
       2     exclude all      0.0.0.0/0  1.1.1.11/32 enable 0
       3     include all      0.0.0.0/0  1.1.1.12/32 enable 1

    Sticky
        Time             : 60
        Overmax          : enable

        --- Option -----------------------------------------------------------
        Src subnet       : 255.255.255.255
        Dst subnet       : 255.255.255.255

    Keep-Backup
        Service          : disable
        Real             : disable

    Backup               :

    Dynamic-Filter       :

    Real
       ID    Name  RIP      Rport Backup Status G-SHDN  Health Cause State-Time      Description
       1     fw1   1.1.1.11              enable disable ACT          0 days 01:24:39
       2     fw2   1.1.1.12              enable disable ACT          0 days 00:47:11

    Health-Check-Info
       ID    Type  Port  Status
       1     icmp  0     enable

    Health-Check-Result
        ID Name Total Active: 1
        1  fw1      O         O (1 ms)
        2  fw2      O         O (1 ms)

    Real Health-Check-Result
        ID Total
        1      D
        2      D

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           1     fw1   0                38
           2     fw2   0                1

        CPS              : 0
        Current sessions : 0
        Total sessions   : 39

    Service-Chain-Info   :

================================================================================
```

## pas-k (int)
- vlan, 인터페이스
```
int_1# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                              |          |
          |         |                   1 1 1 1 1 1 1 |          |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 | TBM      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u . u u . u u u . | disable  |
int       | 10      | . . . . . . . . u . . . . . . t | disable  |
fw1       | 20      | . . . . . . . . . . . u . . . t | disable  |
fw2       | 30      | . . . . . . . . . . . . . . . t | disable  |
================================================================================

int_1# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address     Broadcast       RPF     Description
  fw1     up     00:06:c4:84:10:23 2.2.2.1/32       2.2.2.1         default
  fw2     up     00:06:c4:84:10:23 2.2.2.1/32       2.2.2.1         default
  int     up     00:06:c4:84:10:23 2.2.2.1/24       2.2.2.255       default
  default down   00:06:c4:84:10:23                                  default
  mgmt    up     00:06:c4:84:10:22 192.168.100.1/24 192.168.100.255 default
================================================================================
```

- fwlb 서비스
```
int_1# show info fwlb int

================================================================================
  FWLB: int
 ------------------------------------------------------------------------------
    Name                 : int
    Status               : enable
    Priority             : 100
    LB Method            : rr
    VPNLB                : disable
    Multi Tunnel         : disable
    Branch Relay         : disable
    Position             : internal
    Fail Skip            : none
    Session Timeout Mode : global
    Session Reset        : none
    Session-sync         : none
    H/C Condition        : all
    Health Check         : 1
    Passive Health Check :
    Service Health       : ACT
    Description          :

    Filter
       ID    Type    Protocol Src IP    Dst IP      Status Hit   Description
       1     include all      0.0.0.0/0 1.1.1.0/24  enable 0
       2     exclude all      0.0.0.0/0 1.1.1.11/32 enable 0
       3     exclude all      0.0.0.0/0 1.1.1.12/32 enable 0

    Sticky
        Time             : 60
        Src subnet       : 255.255.255.255
        Dst subnet       : 255.255.255.255

    Keep-Backup
        Service          : disable
        Real             : disable

    Backup               :

    Dynamic-Filter       :

    Real
       ID    Name  RIP      Rport Backup Status G-SHDN  Health Cause State-Time      Description
       1     fw1   2.2.2.11              enable disable ACT          0 days 01:11:45
       2     fw2   2.2.2.12              enable disable ACT          0 days 00:59:46

    Health-Check-Info
       ID    Type  Port  Status
       1     icmp  0     enable

    Health-Check-Result
        ID Name Total Active: 1
        1  fw1      O         O (1 ms)
        2  fw2      O         O (1 ms)

    Real Health-Check-Result
        ID Total
        1      D
        2      D

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           1     fw1   0                0
           2     fw2   0                0

        Current sessions : 0
        Total sessions   : 0

================================================================================
```

## real (서버)
- 인터페이스, vlan
	- management access 8080 open
```
server# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address Broadcast RPF     Description
  lan     up     00:06:c4:84:0c:4b 2.2.2.101/24 2.2.2.255 default
  default down   00:06:c4:84:0c:4b                        default
  mgmt    up     00:06:c4:84:0c:4a                        default
================================================================================

server# sho vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                              |          |
          |         |                   1 1 1 1 1 1 1 |          |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 | TBM      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u . u u u u u u u | disable  |
lan       | 60      | . . . . . . . . u . . . . . . . | disable  |
================================================================================
```

- 라우팅
	- 클라이언트가 다른 대역에 있기 때문에 gw를 통해 나간다
	- 상단의 pask vrrp vip로 지정
```
server# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 2.2.2.250

    Network
       Destination Gateway Interface HC-ID HC-Type HC-Result Description
       2.2.2.0/24  0.0.0.0 lan
================================================================================
```

## FW(방화벽)
- 테스트 구성을 위해 방화벽을 pask 대신 사용
- 실제로는 라우팅 때문에 port boundary 설정이 필요하지만,
  1516 장비 모델 특성으로 인해 포트 바운더리가 없어도 정상적으로 동작함
```
fw1# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                              |          |
          |         |                   1 1 1 1 1 1 1 |          |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 | TBM      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u u u . . u u u u | disable  |
ext       | 20      | . . . . . . . . . . u . . . . . | disable  |
int       | 30      | . . . . . . . . . . . u . . . . | disable  |
================================================================================

fw1#
fw1# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address     Broadcast       RPF     Description
  ext     up     00:06:c4:84:09:b7 1.1.1.11/24      1.1.1.255       default
  int     up     00:06:c4:84:09:b7 2.2.2.11/24      2.2.2.255       default
  default down   00:06:c4:84:09:b7                                  default
  mgmt    up     00:06:c4:84:09:b6 192.168.100.1/24 192.168.100.255 default
================================================================================

fw1#
fw1# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 1.1.1.254

    Network
       Destination Gateway Interface HC-ID HC-Type HC-Result Description
       1.1.1.0/24  0.0.0.0 ext
       2.2.2.0/24  0.0.0.0 int
================================================================================
```
- fw2
```
fw2# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address     Broadcast       RPF     Description
  ext     up     00:06:c4:84:0a:63 1.1.1.12/24      1.1.1.255       default
  int     up     00:06:c4:84:0a:63 2.2.2.12/24      2.2.2.255       default
  default down   00:06:c4:84:0a:63                                  default
  mgmt    up     00:06:c4:84:0a:62 192.168.100.1/24 192.168.100.255 default
================================================================================

fw2# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                              |          |
          |         |                   1 1 1 1 1 1 1 |          |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 | TBM      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u u u . . u u u u | disable  |
ext       | 20      | . . . . . . . . . . u . . . . . | disable  |
int       | 30      | . . . . . . . . . . . u . . . . | disable  |
================================================================================
```

# 동작 확인
- 엔트리 확인
	- entry는 장비에서 하단으로 부하분산 하면서 나가는 순간 기록된다
- fwlb는 라우팅 하지 않고, 설정된 fw 인터페이스로 포워딩 해주는 방식
- ext
```
ext_1# show entry
================================================================================
Prot [Org]Sip:Sport Dip:Dport      - [Rep]Sip:Sport Dip:Dport      Svc:Real   [R]Svc:Real  ND
--------------------------------------------------------------------------------
tcp  1.1.1.50:18528 2.2.2.101:8080 - 2.2.2.101:8080 1.1.1.50:18528 fwlb.ext:2              1
tcp  1.1.1.50:11305 2.2.2.101:8080 - 2.2.2.101:8080 1.1.1.50:11305 fwlb.ext:2              1
================================================================================
```
- int
```
int_2# show entry
================================================================================
Prot [Org]Sip:Sport Dip:Dport      - [Rep]Sip:Sport Dip:Dport      Svc:Real      [R]Svc:Real
--------------------------------------------------------------------------------
tcp  1.1.1.50:18528 2.2.2.101:8080 - 2.2.2.101:8080 1.1.1.50:18528 [R]fwlb.int:2
tcp  1.1.1.50:16006 2.2.2.101:8080 - 2.2.2.101:8080 1.1.1.50:16006 [R]fwlb.int:2
tcp  1.1.1.50:34840 2.2.2.101:8080 - 2.2.2.101:8080 1.1.1.50:34840 [R]fwlb.int:2
tcp  1.1.1.50:11305 2.2.2.101:8080 - 2.2.2.101:8080 1.1.1.50:11305 [R]fwlb.int:2
================================================================================
```

## H/C 동작 확인
- 요청할 때는 h/c tip (보통 vip로) 전송
	- 마스터/백업 모두 자기 인터페이스 ip/mac 사용
	- 목적지는 설정된 tip, mac은 arp 테이블 참고
		- 헬스체크 할 real에 설정된 인터페이스로 보내는 것, 라우팅 동작X
	-  real에서는 라우팅(또는 스위칭) 해서 목적지로 도착
	- 상대편에서는 마스터 장비가 vip/vmac으로 응답 ()
```
ext_2# show real 1

================================================================================
  REAL: 1
 ------------------------------------------------------------------------------
    ID                       : 1
    Name                     : fw1
    Ndomain                  : 1
    RIP                      : 192.168.212.50
    Rport                    :
    SSL Rport                :
    Priority                 : 0
    Weight                   : 1
    Interface                : fw1     <---real에 인터페이스 지정되어 있음
    Site                     :
    MAC Address              :
    Backup                   :
    Graceful Shutdown        : disable
    Manual-Resume            : disable
    SP Filter                :
    SP Filter Group          :
    Domain Filter            :
    Max Connection           : 0
    Max upload-bandwidth     : 0
    Max download-bandwidth   : 0
    Pool Size                : 10000
    Pool Age                 : 3600
    Pool Reuse               : 100
    Pool Src MASK            : 32
    Surge Base Threshold     : 0
    Surge Upper Limit        : 0
    Src NAT IP               :
    Service-IP               :
    Status                   : enable
    Description              :

    Health Check             :
================================================================================

ext_2#
ext_2# show arp

================================================================================
  ARP
 ------------------------------------------------------------------------------
    Timeout (sec)               : 1200
    Locktime (1/100 sec)        : 100
    Proxy Arp Status            : enable
    Proxy Arp Delay (1/100 sec) : 0

    Static
       Ndomain IP Address MAC Address Interface Description
       0
       1

    Dynamic
       Ndomain IP Address     MAC Address       Interface State
       0
       1       192.168.212.10 00:06:c4:94:1c:03 ext       STALE
               192.168.212.50 00:06:c4:84:0a:63 fw1       REACHABLE   <---- 확인
               192.168.212.60 00:06:c4:84:0c:4b ext       STALE
               192.168.212.61 00:0c:29:4e:b9:69 fw2       STALE
================================================================================
```
### H/C 요청
- 마스터
	- 192.168.212.20
	- 0c:4b: fw2
	- 0a:63: fw1
```
ext_2# tcpdump -nei fw1 host 10.10.10.250
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw1, link-type EN10MB (Ethernet), capture size 65535 bytes

13:07:54.151237 00:06:c4:94:1c:0f > 00:06:c4:84:0a:63, ethertype IPv4 (0x0800), length 234: 192.168.212.20 > 10.10.10.250: 
ICMP echo request, id 23799, seq 256, length 200
13:07:54.151634 00:06:c4:84:0a:63 > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 234: 10.10.10.250 > 192.168.212.20: ICMP echo reply, id 23799, seq 256, length 200 

//같은 l2 도메인이기 때문에
10.10.10.250 fw1(백업 하단) mac으로 보내고, fw1에서 arp 테이블을 보고 vmac으로 전송


ext_2# tcpdump -nei fw2 host 10.10.10.250
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw2, link-type EN10MB (Ethernet), capture size 65535 bytes

12:52:02.878346 00:06:c4:94:1c:0f > 00:06:c4:84:0c:4b, ethertype IPv4 (0x0800), length 234: 192.168.212.20 > 10.10.10.250: ICMP echo request, id 18542, seq 256, length 200
12:52:02.878746 00:06:c4:84:0c:4b > 00:06:c4:94:1c:0f, ethertype IPv4 (0x0800), length 234: 10.10.10.250 > 192.168.212.20: ICMP echo reply, id 18542, seq 256, length 200

// 바로 하단에 있는 real

-------------
ext_2# tcpdump -nei fw1 host 192.168.212.250
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw1, link-type EN10MB (Ethernet), capture size 65535 bytes

13:18:17.525120 00:06:c4:84:0a:63 > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 234: 10.10.10.20 > 192.168.212.250: ICMP echo request, id 65239, seq 256, length 200
13:18:17.525157 00:00:5e:00:01:01 > 00:06:c4:84:0a:63, ethertype IPv4 (0x0800), length 234: 192.168.212.250 > 10.10.10.20: ICMP echo reply, id 65239, seq 256, length 200 //h/c에 대한 응답은 vmac/vip로 전송

// 마찬가지로 fw1 하단 방화벽에서 맥러닝이 vmac으로 되어 있어서 vmac을 dst로 전송하는 걸로 보임

ext_2# tcpdump -nei fw2 host 192.168.212.250
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw2, link-type EN10MB (Ethernet), capture size 65535 bytes
13:55:17.367003 00:06:c4:84:0c:4b > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 234: 10.10.10.10 > 192.168.212.250: ICMP echo request, id 37758, seq 256, length 200
13:55:17.367049 00:00:5e:00:01:01 > 00:06:c4:84:0c:4b, ethertype IPv4 (0x0800), length 234: 192.168.212.250 > 10.10.10.10: ICMP echo reply, id 37758, seq 256, length 200
```

- 백업
```
ext_1# tcpdump -nei fw1 host 10.10.10.250
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw1, link-type EN10MB (Ethernet), capture size 65535 bytes
13:03:00.345277 00:06:c4:94:1c:03 > 00:06:c4:84:0a:63, ethertype IPv4 (0x0800), length 234: 192.168.212.10 > 10.10.10.250: ICMP echo request, id 58526, seq 256, length 200
13:03:00.345758 00:06:c4:84:0a:63 > 00:06:c4:94:1c:03, ethertype IPv4 (0x0800), length 234: 10.10.10.250 > 192.168.212.10: ICMP echo reply, id 58526, seq 256, 

ext_1# tcpdump -nei fw2 host 10.10.10.250
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw2, link-type EN10MB (Ethernet), capture size 65535 bytes
13:06:35.633315 00:06:c4:94:1c:03 > 00:06:c4:84:0c:4b, ethertype IPv4 (0x0800), length 234: 192.168.212.10 > 10.10.10.250: ICMP echo request, id 47774, seq 256, length 200
13:06:35.633737 00:06:c4:84:0c:4b > 00:06:c4:94:1c:03, ethertype IPv4 (0x0800), length 234: 10.10.10.250 > 192.168.212.10: ICMP echo reply, id 47774, seq 256, length 200

-----------------
ext_1# tcpdump -nei fw1 host 192.168.212.250
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw1, link-type EN10MB (Ethernet), capture size 65535 bytes
13:07:29.348453 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.212.250 tell 192.168.212.250, length 46
^CExiting...
Done

1 packets captured
1 packets received by filter
0 packets dropped by kernel
ext_1#
ext_1# tcpdump -nei fw2 host 192.168.212.250
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw2, link-type EN10MB (Ethernet), capture size 65535 bytes
13:07:59.348574 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.212.250 tell 192.168.212.250, length 46
^CExiting...
Done

//백업은 H/C 요청에 대해서는 응답하지 않는 걸로 보임
L2구간은 tcpdump에 안잡힘, 하단 fw1에서 올라오는 트래픽을 백업이 스위칭해서 마스터한테 주는 걸로 파악됨

하단 real에서 확인
fw1# tcpdump -nei ext host 192.168.212.250
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ext, link-type EN10MB (Ethernet), capture size 65535 bytes
13:47:46.413970 00:06:c4:84:0a:63 > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 234: 10.10.10.20 > 192.168.212.250: ICMP echo request, id 65239, seq 256, length 200
13:47:46.414182 00:00:5e:00:01:01 > 00:06:c4:84:0a:63, ethertype IPv4 (0x0800), length 234: 192.168.212.250 > 10.10.10.20: ICMP echo reply, id 65239, seq 256, length 200
13:47:48.963396 00:06:c4:84:0a:63 > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 234: 10.10.10.10 > 192.168.212.250: ICMP echo request, id 2547, seq 256, length 200
13:47:48.963575 00:00:5e:00:01:01 > 00:06:c4:84:0a:63, ethertype IPv4 (0x0800), length 234: 192.168.212.250 > 10.10.10.10: ICMP echo reply, id 2547, seq 256, length 200

arp 테이블 확인
fw1# show arp

================================================================================
  ARP
 ------------------------------------------------------------------------------
    Timeout (sec)               : 30
    Locktime (1/100 sec)        : 100
    Proxy Arp Status            : disable
    Proxy Arp Delay (1/100 sec) : 0
    Proxy Arp Running           : disable

    Static                      :

    Dynamic
       IP Address      MAC Address       Interface State
       10.10.10.10     00:06:c4:84:10:27 int       REACHABLE
       10.10.10.20     00:06:c4:84:09:b7 int       REACHABLE
       10.10.10.250    00:00:5e:00:01:02 int       REACHABLE
       192.168.212.10  00:06:c4:94:1c:03 ext       DELAY
       192.168.212.20  00:00:5e:00:01:01 ext       DELAY
       192.168.212.250 00:00:5e:00:01:01 ext       REACHABLE
================================================================================
```

-----------
# proxy arp , arp filter 테스트
## case-행정공제회
- 증상
	- 2호기 장비에서 상/하단 모두 real2만 h/c inact
- 원인
	- 마스터의 proxy arp 동작으로 인해 real 2 서버가 arp를 잘못 학습한 것으로 추정
- 조치
	- 상/하단 마스터, 백업 모두 arp filter 설정 (input drop)
	- 인터링크, real과 연결되는 구간 인터페이스
	- 조치 후 확인 결과 백업 하단 real에서 arping 백업ip 했을 때 마스터로 arp request는 전송되지만 응답하지 않는 걸로 확인됨
		- 필터 없을 경우 백업, 마스터 둘 다 응답. 마스터가 더 늦게 오기 때문에 마스터 mac으로 잘못 학습될 수 있음
### 테스트1
- 백업에서 자기자신으로 arping 테스트 -> 마스터가 proxy arp로 응답
- k1800 이상은 아래와 같이 shell 들어가서 명령어 실행
```
root@ext_2(init_net):~# ns-svc
root@ext_2(svc_net):~# arping -i fw2 192.168.212.21 -S 192.168.212.21
ARPING 192.168.212.21
60 bytes from 00:00:5e:00:01:01 (192.168.212.21): index=0 time=136.137 usec
60 bytes from 00:00:5e:00:01:01 (192.168.212.21): index=1 time=144.005 usec
60 bytes from 00:00:5e:00:01:01 (192.168.212.21): index=2 time=134.945 usec
60 bytes from 00:00:5e:00:01:01 (192.168.212.21): index=3 time=138.044 usec
60 bytes from 00:00:5e:00:01:01 (192.168.212.21): index=4 time=118.971 usec
```

- arp filter 동작 확인
	- arp filter는 mp에만 적용됨, 스위칭은 정상 동작
	- input drop: 설정한 ip에 대한 arp request 요청을 pas-k가 드롭 (cpu에서)
```
ext_2(config)# tcpdump -nei fw1 net 192.168.212.11/32
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw1, link-type EN10MB (Ethernet), capture size 65535 bytes
10:49:01.761023 00:06:c4:94:1c:03 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.212.11 tell 192.168.212.11, length 46
10:49:01.761055 00:06:c4:94:1c:0f > 00:06:c4:94:1c:03, ethertype ARP (0x0806), length 42: Reply 192.168.212.11 is-at 00:00:5e:00:01:01, length 28   <--- 마스터가 vmac을 사용해서 자신이 응답
10:49:02.761029 00:06:c4:94:1c:03 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.212.11 tell 192.168.212.11, length 46
10:49:02.761065 00:06:c4:94:1c:0f > 00:06:c4:94:1c:03, ethertype ARP (0x0806), length 42: Reply 192.168.212.11 is-at 00:00:5e:00:01:01, length 28
10:49:03.761038 00:06:c4:94:1c:03 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.212.11 tell 192.168.212.11, length 46
10:49:03.761077 00:06:c4:94:1c:0f > 00:06:c4:94:1c:03, ethertype ARP (0x0806), length 42: Reply 192.168.212.11 is-at 00:00:5e:00:01:01, length 28
10:49:04.761008 00:06:c4:94:1c:03 > ff:ff:ff:ff:ff:ff, ethertype^CExiting...
Done

8 packets captured
8 packets received by filter
0 packets dropped by kernel

//arp filter 설정 후
ext_2(config)# show arp-filter

================================================================================
  ARP-FILTER
 ------------------------------------------------------------------------------
    Input
       ID    Action Src IP    Dest IP           Interface ND    Description
       1     drop  0.0.0.0/0 192.168.212.11/32 fw1       1

    Output               :
================================================================================


ext_2(config)# tcpdump -nei fw1 net 192.168.212.11/32
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on fw1, link-type EN10MB (Ethernet), capture size 65535 bytes
10:50:38.761141 00:06:c4:94:1c:03 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.212.11 tell 192.168.212.11, length 46
10:50:39.761144 00:06:c4:94:1c:03 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.212.11 tell 192.168.212.11, length 46
10:50:40.761144 00:06:c4:94:1c:03 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.212.11 tell 192.168.212.11, length 46
^CExiting...
Done
```

- 동작 정리
	- proxy arp가 켜져 있고, 라우팅 테이블에 해당 ip에 대한 경로를 가지고 있을 경우 pas-k가 자신의 mac으로 대신 응답한다
		- 요청을 보낸 장비에게 arp reply를 하고 그 이후로는 자신의 라우팅 테이블에 지정된 경로로 트래픽을 전송함
	- **백업 장비는 proxy arp 응답 안함 (자기 자신 ip인 경우 제외)**

### 테스트2
- 백업 하단 real에서 백업ip로 arping 결과
	- 동일 broadcast 구간
```
fw1# arping -i ext 192.168.212.10
ARPING 192.168.212.10
60 bytes from 00:06:c4:94:1c:0f (192.168.212.10): index=0 time=231.981 usec
60 bytes from 00:06:c4:94:1c:03 (192.168.212.10): index=1 time=273.943 usec
^C
--- 192.168.212.10 statistics ---
1 packets transmitted, 2 packets received,   0% unanswered (1 extra)
Done
fw1#
fw1# show arp

================================================================================
  ARP
 ------------------------------------------------------------------------------
    Timeout (sec)               : 30
    Locktime (1/100 sec)        : 100
    Proxy Arp Status            : disable
    Proxy Arp Delay (1/100 sec) : 0
    Proxy Arp Running           : disable

    Static                      :

    Dynamic
       IP Address      MAC Address       Interface State
       10.10.10.10     00:06:c4:84:10:27 int       REACHABLE
       10.10.10.20     00:06:c4:84:09:b7 int       REACHABLE
       10.10.10.250    00:00:5e:00:01:02 int       DELAY
       192.168.212.10  00:00:5e:00:01:01 ext       REACHABLE   <------오염
       192.168.212.20  00:00:5e:00:01:01 ext       REACHABLE
       192.168.212.250 00:00:5e:00:01:01 ext       REACHABLE
================================================================================
```
