# 참고
- AEU는 default gw를 지정하지 않으면 서비스를 타지 않음
	- routing도 하지 않음
- vrrp 서비스는 mp로 처리한다 (포트바운더리 필요x)
- virtual bridge 구성 설명
```
vlan은 인터페이스마다 설정 (3개)
ip 대역 하나만 사용
ip 대역 하나만 사용하기 위해 fw과 연결되는 인터페이스를 /32로 설정한다
```

# 구성
![[image-6.png|659]]

- 방화벽과 연결되는 구간- untag
- 여러 vlan이 지나가야 되는 구간을 tag로 설정해주면 된다
- 현재 구성의 경우 ge1을 통해 서비스

# 동작 요약
- 

# 설정
## pas-k
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