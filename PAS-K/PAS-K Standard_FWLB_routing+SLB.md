# 구성
- slb는 internal 장비에 구성한다
- virtual bridge에서 생성했던 vlan과 동일하지만 이번에는 vlan마다 네트워크 대역을 부여해야 함
- vrrp vip는 마찬가지로 가지고 있는 인터페이스 별로 모두 지정해줘야 한다
![[image-7.png|704]]

# 설정
## pas-k
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

```

- 인터페이스
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
```

- 이중화
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

```

- FWLB
```

```