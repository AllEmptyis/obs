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

# 동작 요약
- 

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
fw        | 20      | t . . . . . . . . . u . . . . . . . . . | . . | disable  | 1       |
fw2       | 30      | t . . . . . . . . . . . . . . . . . . . | . . | disable  | 1       |
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
```

- vrrp
```
ext_1# show failover

================================================================================
  FAILOVER
 ------------------------------------------------------------------------------
    Delay-Time             : 10

    Session-Sync
        Status                     : disable
        Live update interval (10 msec) : 100
        Full update interval (sec) : 30
        Update method              : live
        Peer                       : node2

        Interface
            Name                   :
            IPv4 Address           :
            Peer IP Address        :
            Hc-Retry               : 3
            Health                 : inact

    A-A Failover
        Method             : disable

    Vrrp
       VRID  Mode           Running Status Total Priority VLAN  VIP       VMAC              Description
       1     active-standby master  enable 101   100      fw    1.1.1.250 00:00:5e:00:01:01
                                                          ext   1.1.1.250 00:00:5e:00:01:01
                                                          fw2   1.1.1.250 00:00:5e:00:01:01



```