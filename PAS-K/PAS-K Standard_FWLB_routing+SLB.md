# 구성
- slb는 internal 장비에 구성한다
- virtual bridge에서 생성했던 vlan과 동일하지만 이번에는 vlan마다 네트워크 대역을 부여해야 함
- vrrp vip는 마찬가지로 가지고 있는 인터페이스 별로 모두 지정해줘야 한다
![[image-7.png|704]]

# 설정
## pas-k (EXT)
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
fw1       | 20      | t . . . . . . . . . u . . . . . . . . . | . . | disable  | 1       |
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
fw1       | 20      | t . . . . . . . . . . . . . . . . . . . | . . | disable  |
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
  ext     up     00:06:c4:94:1c:0f 1.1.1.1/24       1.1.1.255       default 1
  fw1     up     00:06:c4:94:1c:0f 2.2.2.1/24       2.2.2.255       default 1
  fw2     up     00:06:c4:94:1c:0f 3.3.3.1/24       3.3.3.255       default 1
  default down   00:06:c4:94:1c:0f                                  default 1
  mgmt    up     00:06:c4:94:1c:0e 192.168.100.1/24 192.168.100.255 default 0
================================================================================

ext_2# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address     Broadcast       RPF     Description
  ext     up     00:06:c4:94:1c:03 1.1.1.2/24       1.1.1.255       default
  fw1     up     00:06:c4:94:1c:03 2.2.2.2/24       2.2.2.255       default
  fw2     up     00:06:c4:94:1c:03 3.3.3.2/24       3.3.3.255       default
  default down   00:06:c4:94:1c:03                                  default
  mgmt    up     00:06:c4:94:1c:02 192.168.100.1/24 192.168.100.255 default
================================================================================
```

- 라우팅
```
ext_1# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 1.1.1.250

    Network
       Destination Gateway Interface Priority HC-ID HC-Type HC-Result Description
       1.1.1.0/24  0.0.0.0 ext
       2.2.2.0/24  0.0.0.0 fw1
       3.3.3.0/24  0.0.0.0 fw2
================================================================================

ext_2# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 1.1.1.250

    Network
       Destination Gateway Interface Priority HC-ID HC-Type HC-Result Description
       1.1.1.0/24  0.0.0.0 ext
       2.2.2.0/24  0.0.0.0 fw1
       3.3.3.0/24  0.0.0.0 fw2
================================================================================
```

- 포트 바운더리
```
ext_1# show port-boundary

================================================================================
PORT-BOUNDARY Configuration
 ------------------------------------------------------------------------------
  ID    Status Type    Boundary Promisc Include MAC Port List    Description
  1     enable include all      off     none        ge1,ge9,ge11
================================================================================

ext_2# show port-boundary

================================================================================
PORT-BOUNDARY Configuration
 ------------------------------------------------------------------------------
  ID    Status Type    Boundary Promisc Include MAC Port List    Description
  1     enable include all      off     none        ge1,ge9,ge11
================================================================================
```

- 이중화 //백업도 동일
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
    interface ext vip 1.1.1.250
    interface ext advertise-send enable
    interface fw1 vip 2.2.2.250
    interface fw1 advertise-send enable
    interface fw2 vip 3.3.3.250
    interface fw2 advertise-send enable
    advertise-interval 10
    retry 3
    arp-count 0
    vmac enable
    preemption enable
    track single-port 1 port ge11
    apply
```

- H/C, real
```
ext_1# show health-check

================================================================================
HEALTH-CHECK Configuration
 ------------------------------------------------------------------------------
  ID    Type  Timeout Interval Status Port  Description
  1     icmp  3       5        enable 0
================================================================================

ext_1# show real

================================================================================
REAL Configuration
 ------------------------------------------------------------------------------
  ID    Name  RIP      Rport SSL-Rport Weight Backup SVC-IP Status Description
  1           2.2.2.11                 1                    enable
  2           3.3.3.11                 1                    enable
================================================================================

ext_2# show real

================================================================================
REAL Configuration
 ------------------------------------------------------------------------------
  ID    Name  RIP      Rport SSL-Rport Weight Backup SVC-IP Status Description
  1           2.2.2.11                 1                    enable
  2           3.3.3.11                 1                    enable
================================================================================

ext_2# show health-check 1

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
    TIP                  : 6.6.6.250
    DIP                  :
    Increase ICMP ID     : disable
================================================================================
```

- FWLB
	- fwlb H/C는 방화벽으로의 ping과는 상관 없음
	- H/C가 검사하는 것은 tip (그렇기 때문에 h/c가 실패한다면 구간 별로 확인 필요)
```
ext_2# show info fwlb ext

================================================================================
  FWLB: ext
 ------------------------------------------------------------------------------
    Name                 : ext
    IP Version           : ipv4
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
       1     include all      0.0.0.0/0 6.6.6.0/24  enable 186
       2     exclude all      0.0.0.0/0 2.2.2.11/32 enable 0
       3     exclude all      0.0.0.0/0 3.3.3.11/32 enable 0

    Sticky
        Time             : 60

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
       1           2.2.2.11              enable disable ACT          0 days 00:49:35
       2           3.3.3.11              enable disable ACT          0 days 01:11:22

    Health-Check-Info
       ID    Type  Port  Status
       1     icmp  0     enable

    Health-Check-Result
        ID Name Total Active: 1
        1           O         O (1 ms)
        2           O         O (1 ms)

    Real Health-Check-Result
        ID Total
        1      D
        2      D

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           1           0                21
           2           0                60

        CPS              : 0
        Current sessions : 0
        Total sessions   : 81

    Service-Chain-Info   :

================================================================================
```

## FW(방화벽)
- 인터페이스, vlan
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
ext       | 10      | . . . . . . . . . . u . . . . . | disable  |
int       | 20      | . . . . . . . . . . . u . . . . | disable  |
================================================================================

fw1# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address     Broadcast       RPF     Description
  ext     up     00:06:c4:84:09:b7 2.2.2.11/24      2.2.2.255       default
  int     up     00:06:c4:84:09:b7 4.4.4.11/24      4.4.4.255       default
  default down   00:06:c4:84:09:b7                                  default
  mgmt    up     00:06:c4:84:09:b6 192.168.100.1/24 192.168.100.255 default
================================================================================
```

- 라우팅
	- gw 잡는 것 주의 - 연결되는 인터페이스의 vrrp vip로 설정
```
fw1# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 2.2.2.250

    Network
       Destination Gateway   Interface HC-ID HC-Type HC-Result Description
       1.1.1.0/24  2.2.2.250 ext
       2.2.2.0/24  0.0.0.0   ext
       4.4.4.0/24  0.0.0.0   int
       6.6.6.0/24  4.4.4.250 int
================================================================================
```

## pas-k (INT)
- vlan, 인터페이스
```
int_2# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                              |          |
          |         |                   1 1 1 1 1 1 1 |          |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 | TBM      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u u u u . u u u . | disable  |
int       | 10      | . . . . . . . . . . . . . . . t | disable  |
fw1       | 20      | . . . . . . . . . . . . . . . t | disable  |
fw2       | 30      | . . . . . . . . . . . u . . . t | disable  |
================================================================================

int_2# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address     Broadcast       RPF     Description
  fw1     up     00:06:c4:84:10:27 4.4.4.2/24       4.4.4.255       default
  fw2     up     00:06:c4:84:10:27 5.5.5.2/24       5.5.5.255       default
  int     up     00:06:c4:84:10:27 6.6.6.2/24       6.6.6.255       default
  default down   00:06:c4:84:10:27                                  default
  mgmt    up     00:06:c4:84:10:26 192.168.100.1/24 192.168.100.255 default
================================================================================
```

- 포트 바운더리
```
int_2# show port-boundary

================================================================================
PORT-BOUNDARY Configuration
 ------------------------------------------------------------------------------
  ID    Status Type    Boundary Promisc Include MAC Port List     Description
  1     enable include all      off     none        ge9,ge12,ge16
================================================================================
```

- 라우팅
```
int_2# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 6.6.6.250

    Network
       Destination Gateway Interface HC-ID HC-Type HC-Result Description
       4.4.4.0/24  0.0.0.0 fw1
       5.5.5.0/24  0.0.0.0 fw2
       6.6.6.0/24  0.0.0.0 int
================================================================================
```

- real, H/C
	- real 3은 SLB
```
int_2# show real

================================================================================
REAL Configuration
 ------------------------------------------------------------------------------
  ID    Name   RIP       Rport SSL-Rport Weight Backup SVC-IP Status Description
  1            4.4.4.12                  1                    enable
  2            5.5.5.12                  1                    enable
  3     server 6.6.6.100 8080            1                    enable
================================================================================

int_2# show health-check

================================================================================
HEALTH-CHECK Configuration
 ------------------------------------------------------------------------------
  ID    Type  Timeout Interval Status Port  Description
  1     icmp  3       5        enable 0
  2     tcp   3       5        enable 8080
================================================================================
```

- FWLB
	- **internal 장비는 include 필터에 sip를 반드시 넣어야 함 - 왜 안넣었는데 정상 동작하는지는 추가 확인 필요**
		- 이유: internal 장비는 필터의 sip 와 실제 패킷의 dip를 보고 매칭될 때 리버스 엔트리 생성, 그 후 엔트리를 보고 FW 경로 결정
		  external 장비는 반대
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
       1     include all      0.0.0.0/0 1.1.1.0/24  enable 44
       2     exclude all      0.0.0.0/0 4.4.4.11/32 enable 0
       3     exclude all      0.0.0.0/0 5.5.5.11/32 enable 0

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
       1           4.4.4.11              enable disable ACT          0 days 01:09:45
       2           5.5.5.11              enable disable ACT          0 days 02:05:40

    Health-Check-Info
       ID    Type  Port  Status
       1     icmp  0     enable

    Health-Check-Result
        ID Name Total Active: 1
        1           O         O (1 ms)
        2           O         O (1 ms)

    Real Health-Check-Result
        ID Total
        1      D
        2      D

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           1           0                15
           2           0                29

        Current sessions : 0
        Total sessions   : 44

================================================================================
```

- SLB 서비스
```
int_1# show info slb test

================================================================================
  SLB: test
 ------------------------------------------------------------------------------
    Name                 : test
    IP Version           : ipv4
    Status               : enable
    Priority             : 50
    NAT Mode             : dnat
    LB Method            : rr
    Fail Skip            : none
    Fail Action          : default
    Session Timeout Mode : global
    Session Reset        : none
    Session-sync         : none
    H/C Condition        : all
    Health Check         : 2
    Passive Health Check :
    Service Health       : ACT

    Vip
       VIP       Protocol Vport
       6.6.6.101 tcp      80

    Description          :

    Filter
       ID    Type    Protocol Src IP    Dst IP       Status Hit   Description
       1     include tcp      0.0.0.0/0 6.6.6.101/32 enable 18

    Sticky
        Time             : 60

        --- Option -----------------------------------------------------------
        Src Subnet       : 255.255.255.255

    Keep-Backup
        Service          : disable
        Real             : disable

    Backup               :

    Dynamic-Filter       :

    Real
       ID    Name   RIP       Rport Backup Status G-SHDN  Health Cause State-Time      Description
       3     server 6.6.6.100 8080         enable disable ACT          0 days 00:34:54

    Health-Check-Info
       ID    Type  Port  Status
       2     tcp   8080  enable

    Health-Check-Result
        ID Name   Total Active: 2
        3  server     O         O (0 ms)

    Real Health-Check-Result
        ID Name   Total 2
        3  server     O O (1 ms)

    Statistics
        Real
           ID    Name   Current sessions Total sessions
           3     server 0                18

        Current sessions : 0
        Total sessions   : 18

================================================================================
```


## 동작 확인
- vip:vport인 6.6.6.100:80으로 접속
- external
```
ext_2# show entry
================================================================================
Prot [Org]Sip:Sport Dip:Dport    - [Rep]Sip:Sport Dip:Dport      Svc:Real   [R]Svc:Real
--------------------------------------------------------------------------------
tcp  1.1.1.50:38412 6.6.6.101:80 - 6.6.6.101:80   1.1.1.50:38412 fwlb.ext:1
tcp  1.1.1.50:48662 6.6.6.101:80 - 6.6.6.101:80   1.1.1.50:48662 fwlb.ext:1
tcp  1.1.1.50:64907 6.6.6.101:80 - 6.6.6.101:80   1.1.1.50:64907 fwlb.ext:1
tcp  1.1.1.50:4082  6.6.6.101:80 - 6.6.6.101:80   1.1.1.50:4082  fwlb.ext:1
================================================================================
```

- internal
```
int_1# show entry
================================================================================
Prot [Org]Sip:Sport Dip:Dport    - [Rep]Sip:Sport Dip:Dport      Svc:Real   [R]Svc:Real
--------------------------------------------------------------------------------
tcp  1.1.1.50:41171 6.6.6.101:80 - 6.6.6.100:8080 1.1.1.50:41171 slb.test:3 [R]fwlb.int:1
tcp  1.1.1.50:4082  6.6.6.101:80 - 6.6.6.100:8080 1.1.1.50:4082  slb.test:3 [R]fwlb.int:1
tcp  1.1.1.50:64907 6.6.6.101:80 - 6.6.6.100:8080 1.1.1.50:64907 slb.test:3 [R]fwlb.int:1
================================================================================
```

# transparent 구성
- FW가 ip가 없는 브리지 형태인 경우
- 이러한 경우에는 real을 각각 상/하단의 인터페이스 ip로 설정한다

- 주의
	- H/C TIP는 vrrp vip가 아닌 각 인터페이스 ip로 설정
		- FWLB H/C은 H/C에 설정된 tip의 ip와 real의 mac으로 전송
		- FW가 라우팅을 해주는 구성에서는 real의 mac이 FW이기 때문에 상관없지만,
		  
	- 브릿지 구성이기 때문에 어차피 인터페이스 mac을 알아서 굳이 vrrp vip로 잡지 않아도 됨

- vrrp advertisement 광고 규칙
	- advertisement 패킷은 마스터가 전송 (백업으로 패킷이 들어와도 vrrp 응답은 마스터가 응답)
