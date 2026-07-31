# SLB routing 이중화 구성
- 구성
	- standard ppt 참고
## 기본 설정
- vlan
	- 인터링크 부분은 lan에만 tagged로 지정 (real과 직결이 아닌 여러 vlan을 거쳐서 오는 등)
		- 환경에 따라 untag로만 써도 됨
```
SLB1(config)# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                                      | xg  |          |         |
          |         |                   1 1 1 1 1 1 1 1 1 1 2 |     |          |         |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 | 1 2 | TBM      | ND      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u . . . u u u u u u u u u | u u | disable  | 1       |
wan       | 10      | . . . . . . . . . . u . . . . . . . . . | . . | disable  | 1       |
lan       | 20      | . . . . . . . . t u . . . . . . . . . . | . . | disable  | 1       |
================================================================================
```
- 라우팅
```
SLB1(config)# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 192.168.193.1

    Network
       Destination      Gateway Interface Priority HC-ID HC-Type HC-Result Description
       3.3.3.0/24       0.0.0.0 lan
       192.168.193.0/24 0.0.0.0 wan
================================================================================
```
- 인터페이스
```
SLB1(config)# show int lan

================================================================================
  INTERFACE: lan
 ------------------------------------------------------------------------------
    Name                 : lan
    Status               : up
    MAC Address          : 00:06:c4:92:02:f3
    MTU                  : 1500

    IP
       IPv4 Address Broadcast Overlapped
       3.3.3.50/24  3.3.3.255

    IPv6                 :

    RPF                  : default
    ARP-ignore           : 0
    ARP-announce         : 0
    Network Domain       : 1
    Description          :

    --- Option ---------------------------------------------------------------
    Adv-Send-Advert      : enable
    Adv-Default-Lifetime : 1800
    Min-Rtr-Adv-Interval : 198
    Max-Rtr-Adv-Interval : 600
    Adv-Cur-Hop-Limit    : 64
    Adv-Reachable-Time   : 0
    Adv-Retrans-Timer    : 0
================================================================================


SLB1(config)# show int wan

================================================================================
  INTERFACE: wan
 ------------------------------------------------------------------------------
    Name                 : wan
    Status               : up
    MAC Address          : 00:06:c4:92:02:f3
    MTU                  : 1500

    IP
       IPv4 Address      Broadcast       Overlapped
       192.168.193.50/24 192.168.193.255

    IPv6                 :

    RPF                  : default
    ARP-ignore           : 0
    ARP-announce         : 0
    Network Domain       : 1
    Description          :

    --- Option ---------------------------------------------------------------
    Adv-Send-Advert      : enable
    Adv-Default-Lifetime : 1800
    Min-Rtr-Adv-Interval : 198
    Max-Rtr-Adv-Interval : 600
    Adv-Cur-Hop-Limit    : 64
    Adv-Reachable-Time   : 0
    Adv-Retrans-Timer    : 0
================================================================================
```
## real 설정 (real1)
- management access http 활성화
- **GW는 pask의 vrrp vip로 잡아야 한다**
	- 그렇지 않으면 백업의 게이트웨이를 타고 감
```
switch(config)# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 3.3.3.250

    Network
       Destination Gateway Interface Priority HC-ID HC-Type HC-Result Description
       3.3.3.0/24  0.0.0.0 lan
================================================================================

switch#
switch# show int lan

================================================================================
  INTERFACE: lan
 ------------------------------------------------------------------------------
    Name                 : lan
    Status               : up
    MAC Address          : 00:06:c4:94:1b:fb
    MTU                  : 1500

    IP
       IPv4 Address Broadcast Overlapped
       3.3.3.101/24 3.3.3.255

    IPv6                 :

    RPF                  : default
    ARP-ignore           : 0
    ARP-announce         : 0
    Network Domain       : 1
    Description          :

    --- Option ---------------------------------------------------------------
    Adv-Send-Advert      : enable
    Adv-Default-Lifetime : 1800
    Min-Rtr-Adv-Interval : 198
    Max-Rtr-Adv-Interval : 600
    Adv-Cur-Hop-Limit    : 64
    Adv-Reachable-Time   : 0
    Adv-Retrans-Timer    : 0
================================================================================
```

## slb 설정
- slb 설정
- vip-192.168.193.130 : 80
- rip-3.3.3.101, 3.3.3.102:8080
```
SLB1(config)# show info slb test

================================================================================
  SLB: test
 ------------------------------------------------------------------------------
    Name                 : test
    IP Version           : ipv4
    Ndomain              : 1
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
    Health Check         : 1
    Passive Health Check :
    Service Health       : ACT

    Vip
       VIP             Protocol Vport
       192.168.193.130 tcp      80

    Description          :

    Filter
       ID    Type    Protocol Src IP    Dst IP             Status Hit   Description
       1     include tcp      0.0.0.0/0 192.168.193.130/32 enable 199

    Sticky
        Time             : 60
        Overmax          : enable

        --- Option -----------------------------------------------------------
        Src Subnet       : 255.255.255.255
        Dst Subnet       : 0.0.0.0

    Keep-Backup
        Service          : disable
        Real             : disable

    Backup               :

    Dynamic-Filter       :

    Real
       ID    Name  RIP       Rport Backup Status G-SHDN  Health Cause State-Time      Description
       1     real1 3.3.3.101 8080         enable disable ACT          0 days 04:36:55
       2     real2 3.3.3.102 8080         enable disable ACT          0 days 04:36:35

    Health-Check-Info
       ID    Type  Port  Status
       1     tcp   8080  enable

    Health-Check-Result
        ID Name  Total Active: 1
        1  real1     O         O (0 ms)
        2  real2     O         O (0 ms)

    Real Health-Check-Result
        ID Name  Total 1
        1  real1     O O (0 ms)
        2  real2     O O (0 ms)

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           1     real1 0                4
           2     real2 7                211

        CPS              : 1
        Current sessions : 7
        Total sessions   : 215

    Service-Chain-Info   :

================================================================================
```
- real
```
SLB1(config)# show real

================================================================================
REAL Configuration
 ------------------------------------------------------------------------------
  ID    Name  RIP       Rport SSL-Rport Weight Backup SVC-IP Status Description
  1     real1 3.3.3.101 8080            1                    enable
  2     real2 3.3.3.102 8080            1                    enable
================================================================================
```
- real 1
```
SLB1(config)# show real 1

================================================================================
  REAL: 1
 ------------------------------------------------------------------------------
    ID                       : 1
    Name                     : real1
    Ndomain                  : 1
    RIP                      : 3.3.3.101
    Rport                    : 8080
    SSL Rport                :
    Priority                 : 0
    Weight                   : 1
    Interface                :
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

    Health Check             : 1
================================================================================
```
- health check
```
SLB1(config)# show health-check 1

================================================================================
  HEALTH-CHECK: 1
 ------------------------------------------------------------------------------
    ID                   : 1
    Type                 : tcp
    Timeout              : 3
    Interval             : 5
    Retry                : 3
    Recover              : 0
    Status               : enable
    Graceful Shutdown    : disable
    Description          :

    --- Option ---------------------------------------------------------------
    SIP                  :
    TIP                  :
    DIP                  :
    Port                 : 8080
    Half Open            : disable
    Send                 :
    Expect               :
    Unexpect             :
    Source port min      : 10000
    Source port max      : 65535
================================================================================
```
- port boundary
	- 인바운드 기준으로 서비스 타는 포트는 다 지정해야 함
```
SLB1(config)# show port-boundary

================================================================================
PORT-BOUNDARY Configuration
 ------------------------------------------------------------------------------
  ID    Status Type    Boundary Promisc Include MAC Port List     Description
  1     enable include all      off     none        ge9,ge10,ge11
================================================================================
```

## H/C 실패 경우
- health가 INACT인 경우 > 실패
- unknown인 경우 > 체크 중
```
  Real
       ID    Name  RIP       Rport Backup Status G-SHDN  Health  Cause      State-Time      Description
       1     real1 3.3.3.101 8080         enable disable INACT   HC_TIMEOUT 0 days 00:06:53
       2     real2 3.3.3.102 8080         enable disable UNKNOWN            0 days 00:00:00
```
- TCP CONN_REFUSED
	- 서버 측에서 RST 전송
		- 이유: management access 에서 http enable 활성화 안함
```
   Real
       ID    Name  RIP       Rport Backup Status G-SHDN  Health Cause            State-Time      Description
       1     real1 3.3.3.101 8080         enable disable INACT  TCP_CONN_REFUSED 0 days 01:06:02
       2     real2 3.3.3.102 8080         enable disable INACT  TCP_CONN_REFUSED 0 days 00:58:51
```

## vrrp(이중화)
- active-standby 구성
	- active-active는 거의 사용 안함
- track port가 다운되면 무조건 백업으로 전환된다
- fail over 동작
	- priority가 같을 땐 먼저 선점하는 장비가 마스터가 됨
		- 마스터였던 장비가 다운됐다가 다시 복구되는 경우 > 자동 페일오버 안넘어감
- 마스터인 장비는 mac table 갱신을 위해 garp를 15초 간격으로 전송
```
SLB1(config)# tcpdump -nei lan arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
15:02:05.933319 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.250 tell 3.3.3.250, length 28
15:02:20.933116 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.250 tell 3.3.3.250, length 28
15:02:35.932842 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.250 tell 3.3.3.250, length 28
15:02:50.932583 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.250 tell 3.3.3.250, length 28
15:03:05.932300 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.250 tell 3.3.3.250, length 28
15:03:20.932042 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.250 tell 3.3.3.250, length 28
15:03:35.931806 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.250 tell 3.3.3.250, length 28
```
- vrrp
	- 1초 주기로 광고, 3번 이상 못받으면 failover
	- vrrp 광고  패킷은 자신의 인터페이스로 보낸다
```
13:41:46.207087 00:06:c4:92:02:f3 > 01:00:5e:00:00:f0, ethertype IPv4 (0x0800), length 74: 3.3.3.50 > 224.0.0.240: VRRPv2, Advertisement, vrid 1, prio 102, authtype none, intvl 1s, length 40
13:41:47.207116 00:06:c4:92:02:f3 > 01:00:5e:00:00:f0, ethertype IPv4 (0x0800), length 74: 3.3.3.50 > 224.0.0.240: VRRPv2, Advertisement, vrid 1, prio 102, authtype none, intvl 1s, length 40
13:41:48.207126 00:06:c4:92:02:f3 > 01:00:5e:00:00:f0, ethertype IPv4 (0x0800), length 74: 3.3.3.50 > 224.0.0.240: VRRPv2, Advertisement, vrid 1, prio 102, authtype none, intvl 1s, length 40

13:41:51.306775 00:06:c4:94:1c:17 > 01:00:5e:00:00:f0, ethertype IPv4 (0x0800), length 74: 3.3.3.52 > 224.0.0.240: VRRPv2, Advertisement, vrid 1, prio 102, authtype none, intvl 1s, length 40
```
### 이중화 설정
- slb 1 (백업)
```
SLB1(config)# show failover

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
       VRID  Mode           Running Status Total Priority VLAN  VIP             VMAC              Description
       1     active-standby backup  enable 102   101      lan   3.3.3.250       00:00:5e:00:01:01
                                                          wan   192.168.193.250 00:00:5e:00:01:01
```
- slb 2 (마스터)
```
slb2(config)# show failover

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
       VRID  Mode           Running Status Total Priority VLAN  VIP             VMAC              Description
       1     active-standby master  enable 102   101      lan   3.3.3.250       00:00:5e:00:01:01
                                                          wan   192.168.193.250 00:00:5e:00:01:01
```
- failover 설정
```
slb2(config-failover)# show vrrp 1

================================================================================
  VRRP: 1
 ------------------------------------------------------------------------------
    VRID                          : 1
    Ndomain                       : 1
    Mode                          : active-standby
    Status                        : enable
    Priority                      : 101
    Service VIP                   :
    Send GARP All SVIP            : disable
    Port boundary                 :
    Port-Block                    :

    Interface
       VLAN  Advertise VIP             VMAC              Overlapped
       lan   enable    3.3.3.250       00:00:5e:00:01:01
       wan   enable    192.168.193.250 00:00:5e:00:01:01

    Advertise Interval (100 msec) : 10
    Retry                         : 3
    Arp Count                     : 0
    VMAC                          : enable
    Preemption                    : enable
    Description                   :

    Track
        Member port               :

        Single Port
           Priority Port
           1        ge11

        Track real                :
================================================================================

SLB1(config-failover)# show vrrp 1

================================================================================
  VRRP: 1
 ------------------------------------------------------------------------------
    VRID                          : 1
    Ndomain                       : 1
    Mode                          : active-standby
    Status                        : enable
    Priority                      : 101
    Service VIP                   :
    Send GARP All SVIP            : disable
    Port boundary                 :
    Port-Block                    :

    Interface
       VLAN  Advertise VIP             VMAC              Overlapped
       lan   enable    3.3.3.250       00:00:5e:00:01:01
       wan   enable    192.168.193.250 00:00:5e:00:01:01

    Advertise Interval (100 msec) : 10
    Retry                         : 3
    Arp Count                     : 0
    VMAC                          : enable
    Preemption                    : enable
    Description                   :

    Track
        Member port               :

        Single Port
           Priority Port
           1        ge11

        Track real                :
================================================================================
```

## slb 동작 확인
- vip 192.168.193.130:80으로 접속 시 엔트리 확인 (dnat만 수행)
	- 3.3.3.102로 dnat 해서 전송
	- 다시 클라이언트에게 돌려줄 때는 vip로 snat해서 전송 
```
slb2(config)# show entry
================================================================================
Prot [Org]Sip:Sport        Dip:Dport          - [Rep]Sip:Sport Dip:Dport             Svc:Real   [R]Svc:Real  ND
--------------------------------------------------------------------------------
tcp  192.168.212.182:13136 192.168.193.130:80 - 3.3.3.102:8080 192.168.212.182:13136 slb.test:2              1
tcp  192.168.212.182:43391 192.168.193.130:80 - 3.3.3.102:8080 192.168.212.182:43391 slb.test:2              1
tcp  192.168.212.182:59711 192.168.193.130:80 - 3.3.3.102:8080 192.168.212.182:59711 slb.test:2              1
tcp  192.168.212.182:56425 192.168.193.130:80 - 3.3.3.102:8080 192.168.212.182:56425 slb.test:2              1
tcp  192.168.212.182:5512  192.168.193.130:80 - 3.3.3.102:8080 192.168.212.182:5512  slb.test:2              1
tcp  192.168.212.182:11755 192.168.193.130:80 - 3.3.3.102:8080 192.168.212.182:11755 slb.test:2              1
================================================================================
```

- pas-k wan 인터페이스
	- `00:00:5e:00:01:01` vip 요청이 vmac으로 전송
	- `1c:17` pas-k mac
```
slb2(config)# tcpdump -nei wan host 192.168.212.182
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wan, link-type EN10MB (Ethernet), capture size 65535 bytes
16:56:40.343211 00:d0:f8:22:35:42 > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 66: 192.168.212.182.39808 > 192.168.193.130.80: Flags [S], seq 2024876557, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0
16:56:40.343563 00:06:c4:94:1c:17 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 66: 192.168.193.130.80 > 192.168.212.182.39808: Flags [S.], seq 488659058, ack 2024876558, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 8], length 0
```

-  lan 인터페이스
	- pask mac으로 바뀌어서 서버에게 전달 후 서버로부터 응답 받음
```
slb2(config)# tcpdump -nei  lan host 192.168.212.182
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
17:01:58.953209 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype IPv4 (0x0800), length 66: 192.168.212.182.38969 > 3.3.3.102.8080: Flags [S], seq 480331739, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0
17:01:58.953410 00:06:c4:94:1c:0f > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 66: 3.3.3.102.8080 > 192.168.212.182.38969: Flags [S.], seq 3460002548, ack 480331740, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 8], length 0
```

- 서버에서 확인
```
switch2# tcpdump -nei lan host 192.168.212.182
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
17:02:57.329929 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype IPv4 (0x0800), length 531: 192.168.212.182.50362 > 3.3.3.102.8080: Flags [P.], seq 2537139978:2537140455, ack 1706382481, win 255, length 477
17:02:57.330265 00:06:c4:94:1c:0f > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 54: 3.3.3.102.8080 > 192.168.212.182.50362: Flags [.], ack 477, win 251, length 0
```

- 서버 arp 테이블 확인
```
switch2# show arp

================================================================================
  ARP
 ------------------------------------------------------------------------------
    Timeout (sec)               : 1200
    Locktime (1/100 sec)        : 100
    Proxy Arp Status            : disable
    Proxy Arp Delay (1/100 sec) : 0

    Static
       Ndomain IP Address MAC Address Interface Description
       0
       1

    Dynamic
       Ndomain IP Address      MAC Address       Interface State
       0
       1       3.3.3.50        00:06:c4:92:02:f3 lan       REACHABLE
               3.3.3.52        00:00:5e:00:01:01 lan       REACHABLE <--------- 마스터인 slb2의 mac이 vmac으로 학습되어 있다 (현재 3.3.3.250이 gw)
               3.3.3.250       00:00:5e:00:01:01 lan       REACHABLE
               192.168.193.150 00:06:c4:94:1c:17 lan       STALE
               192.168.193.152 00:00:5e:00:01:01 lan       STALE
================================================================================
```

- 서버에서 arp 덤프 확인
	- gw(vrrp vip)를 물어보자 pask가 vmac을 알려준다
	- 즉 vrrp vip로 물어봤기 때문에 vmac을 알 수 있는 것
		- 이후 vmac으로 트래픽 전달
```
switch(config)# tcpdump -nei lan arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
17:08:46.988489 00:06:c4:94:1b:fb > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.250 tell 3.3.3.101, length 28
17:08:46.988623 00:06:c4:94:1c:17 > 00:06:c4:94:1b:fb, ethertype ARP (0x0806), length 60: Reply 3.3.3.250 is-at 00:00:5e:00:01:01, length 46
17:08:47.517027 00:06:c4:94:1b:fb > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.50 tell 3.3.3.101, length 28
17:08:47.517136 00:06:c4:92:02:f3 > 00:06:c4:94:1b:fb, ethertype ARP (0x0806), length 60: Reply 3.3.3.50 is-at 00:06:c4:92:02:f3, length 46
```

- 서버2에서 arp 덤프 확인 (마스터와 직결)
```
switch2(config)# no arp dynamic
switch2(config)# tcpdump -nei lan arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
17:12:15.612407 00:06:c4:94:1c:0f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.52 tell 3.3.3.102, length 28
17:12:15.612534 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype ARP (0x0806), length 60: Reply 3.3.3.52 is-at 00:00:5e:00:01:01, length 46 // 마스터 인터페이스에 대한 arp 응답은 항상 vmac
17:12:15.777540 00:06:c4:94:1c:0f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.50 tell 3.3.3.102, length 28
17:12:15.777635 00:06:c4:92:02:f3 > 00:06:c4:94:1c:0f, ethertype ARP (0x0806), length 60: Reply 3.3.3.50 is-at 00:06:c4:92:02:f3, length 46 // 백업 인터페이스는 실제 mac으로 응답
17:12:19.025245 00:06:c4:94:1c:0f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.250 tell 3.3.3.102, length 28
17:12:19.025358 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype ARP (0x0806), length 60: Reply 3.3.3.250 is-at 00:00:5e:00:01:01, length 46
```

## 이중화 동작 확인
- 마스터(slb2)의 ge11(트랙포트)를 다운시켰을 때
- 서버
	- 백업-마스터 전환 후 마스터가 garp 패킷을 보내서 인접 장비의 mac 테이블을 갱신 
```
switch2(config)# tcpdump -nei lan arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
17:16:39.992870 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 3.3.3.250 tell 3.3.3.250, length 46
17:16:46.001028 00:06:c4:94:1c:17 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 3.3.3.52 tell 3.3.3.52, length 46
17:16:49.092534 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 3.3.3.250 tell 3.3.3.250, length 46
17:16:49.092634 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 3.3.3.250 tell 3.3.3.250, length 46
17:16:50.086556 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 3.3.3.250 tell 3.3.3.250, length 46
17:16:56.008967 00:06:c4:94:1c:0f > 00:06:c4:94:1c:17, ethertype ARP (0x0806), length 42: Request who-has 3.3.3.52 tell 3.3.3.102, length 28
17:16:56.009089 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype ARP (0x0806), length 60: Reply 3.3.3.52 is-at 00:06:c4:94:1c:17, length 46 //백업으로 변경 후 실제 mac으로 응답
17:17:05.086159 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 3.3.3.250 tell 3.3.3.250, length 46 //마스터로 변경된 장비가 mac 테이블 갱신을 위해 보내는 garp
```
# Virtual Bridge SLB 이중화 구성
## 확인
### 오류 - 인터페이스 ip를 vip로 지정한 경우
- 리얼서버와 통신하는 lan 인터페이스 ip를 vip로 지정함
```
SLB1# show int lan

================================================================================
  INTERFACE: lan
 ------------------------------------------------------------------------------
    Name                 : lan
    Status               : up
    MAC Address          : 00:06:c4:92:02:f3
    MTU                  : 1500

    IP
       IPv4 Address       Broadcast       Overlapped
       192.168.193.130/32 192.168.193.130

    IPv6                 :

    RPF                  : default
    ARP-ignore           : 0
    ARP-announce         : 0
    Network Domain       : 1
    Description          :

    --- Option ---------------------------------------------------------------
    Adv-Send-Advert      : enable
    Adv-Default-Lifetime : 1800
    Min-Rtr-Adv-Interval : 198
    Max-Rtr-Adv-Interval : 600
    Adv-Cur-Hop-Limit    : 64
    Adv-Reachable-Time   : 0
    Adv-Retrans-Timer    : 0
================================================================================
```
- 결과
	- PAS-K가 RST을 보낸다, 왜냐하면 **인터페이스 ip는 pask가 받는 순간 mpi로 처리됨 (ppi, 즉 포트 바운더리를 타지 않음)**
	  따라서 RST을 보냄

### 클라이언트가 직접 rip로 통신이 가능한지?
- 최초 통신은 불가, 그러나 proxy arp를 통해 vip로 통신 후 mac learning이 된 후에는 가능하다.
### host routing 설정하지 않은 경우
- pask에서 받기는 하지만 lan 인터페이스로 넘겨주지 못함
```
SLB1(config)# tcpdump -nei wan host 192.168.212.182
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wan, link-type EN10MB (Ethernet), capture size 65535 bytes
09:41:36.536868 00:d0:f8:22:35:42 > 00:06:c4:92:02:f3, ethertype IPv4 (0x0800), length 66: 192.168.212.182.17521 > 192.168.193.130.80: Flags [S], seq 3970014047, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0
09:41:36.536869 00:d0:f8:22:35:42 > 00:06:c4:92:02:f3, ethertype IPv4 (0x0800), length 66: 192.168.212.182.17072 > 192.168.193.130.80: Flags [S], seq 713995550, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0
09:41:36.795899 00:d0:f8:22:35:42 > 00:06:c4:92:02:f3, ethertype IPv4 (0x0800), length 66: 192.168.212.182.48633 > 192.168.193.130.80: Flags [S], seq 1522224016, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0

-----
SLB1(config)# tcpdump -nei lan host 192.168.212.182
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes


^CExiting...
```

### proxy arp 설정하지 않은 경우
- pask에서 리얼서버까지는 정상적으로 통신
- 리얼서버에서 응답을 하려고 할 때 클라이언트의 mac을 모르기 때문에 arp를 보냄, 그러나 pask에서 proxy arp가 비활성화 되어 있으면 arp 응답 불가
- proxy arp가 설정되어 있지 않은 경우 리얼서버에서 gw로도 통신 불가 
	- 왜냐하면 pask가 proxy arp로 응답해줘야 gw도 알아낼 수 있음
## pas-k 설정
- vlan
```
slb2(config)# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                                      | xg  |          |         |
          |         |                   1 1 1 1 1 1 1 1 1 1 2 |     |          |         |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 | 1 2 | TBM      | ND      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u . . . u u u u u u u u u | u u | disable  | 1       |
wan       | 10      | . . . . . . . . . . u . . . . . . . . . | . . | disable  | 1       |
lan       | 20      | . . . . . . . . u u . . . . . . . . . . | . . | disable  | 1       |
================================================================================
```
- 인터페이스 ip
```
slb2(config)# show int wan

================================================================================
  INTERFACE: wan
 ------------------------------------------------------------------------------
    Name                 : wan
    Status               : up
    MAC Address          : 00:06:c4:94:1c:17
    MTU                  : 1500

    IP
       IPv4 Address      Broadcast       Overlapped
       192.168.193.52/24 192.168.193.255

    IPv6                 :

    RPF                  : default
    ARP-ignore           : 0
    ARP-announce         : 0
    Network Domain       : 1
    Description          :

    --- Option ---------------------------------------------------------------
    Adv-Send-Advert      : enable
    Adv-Default-Lifetime : 1800
    Min-Rtr-Adv-Interval : 198
    Max-Rtr-Adv-Interval : 600
    Adv-Cur-Hop-Limit    : 64
    Adv-Reachable-Time   : 0
    Adv-Retrans-Timer    : 0
================================================================================

slb2(config)# show int lan

================================================================================
  INTERFACE: lan
 ------------------------------------------------------------------------------
    Name                 : lan
    Status               : up
    MAC Address          : 00:06:c4:94:1c:17
    MTU                  : 1500

    IP
       IPv4 Address       Broadcast       Overlapped
       192.168.193.152/32 192.168.193.152

    IPv6                 :

    RPF                  : default
    ARP-ignore           : 0
    ARP-announce         : 0
    Network Domain       : 1
    Description          :

    --- Option ---------------------------------------------------------------
    Adv-Send-Advert      : enable
    Adv-Default-Lifetime : 1800
    Min-Rtr-Adv-Interval : 198
    Max-Rtr-Adv-Interval : 600
    Adv-Cur-Hop-Limit    : 64
    Adv-Reachable-Time   : 0
    Adv-Retrans-Timer    : 0
================================================================================
```
- arp
	- arp 삭제: no arp dynamic
```
slb2(config)# show arp

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
       Ndomain IP Address      MAC Address       Interface State
       0
       1       2.2.2.250       00:00:5e:00:01:01 lan       STALE
               172.31.0.1      00:06:c4:71:96:04 wan       STALE
               192.168.193.1   00:00:5e:00:01:c1 wan       REACHABLE
               192.168.193.2   00:06:c4:62:ae:7e wan       STALE
               192.168.193.3   00:d0:f8:22:35:42 wan       STALE
               192.168.193.50  00:06:c4:92:02:f3 wan       STALE
               192.168.193.131 00:06:c4:94:1b:fb lan       REACHABLE
               192.168.193.132 00:06:c4:94:1c:0f lan       REACHABLE
               192.168.193.150 00:06:c4:92:02:f3 lan       STALE
               192.168.193.231 00:06:c4:94:1c:0b wan       STALE
================================================================================
```
- 라우팅
```
slb2(config)# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 192.168.193.1

    Network
       Destination        Gateway Interface Priority HC-ID HC-Type HC-Result Description
       192.168.193.131/32 0.0.0.0 lan
       192.168.193.132/32 0.0.0.0 lan
       192.168.193.0/24   0.0.0.0 wan
================================================================================
```
- slb 설정
```
slb2(config)# show info slb test

================================================================================
  SLB: test
 ------------------------------------------------------------------------------
    Name                 : test
    IP Version           : ipv4
    Ndomain              : 1
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
    Health Check         : 1
    Passive Health Check :
    Service Health       : ACT

    Vip
       VIP             Protocol Vport
       192.168.193.130 tcp      80

    Description          :

    Filter
       ID    Type    Protocol Src IP    Dst IP             Status Hit   Description
       1     include tcp      0.0.0.0/0 192.168.193.130/32 enable 59

    Sticky
        Time             : 60
        Overmax          : enable

        --- Option -----------------------------------------------------------
        Src Subnet       : 255.255.255.255
        Dst Subnet       : 0.0.0.0

    Keep-Backup
        Service          : disable
        Real             : disable

    Backup               :

    Dynamic-Filter       :

    Real
       ID    Name  RIP             Rport Backup Status G-SHDN  Health Cause State-Time      Description
       3           192.168.193.131 8080         enable disable ACT          0 days 00:09:47
       4           192.168.193.132 8080         enable disable ACT          0 days 00:09:42

    Health-Check-Info
       ID    Type  Port  Status
       1     tcp   8080  enable

    Health-Check-Result
        ID Name Total Active: 1
        3           O         O (0 ms)
        4           O         O (0 ms)

    Real Health-Check-Result
        ID Total
        3      D
        4      D

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           3           0                6
           4           0                0

        CPS              : 0
        Current sessions : 0
        Total sessions   : 6

    Service-Chain-Info   :

================================================================================
```
## 리얼 설정
- 라우팅
	- 게이트웨이는 같은 vlan이기 때문에 상단 게이트웨이 잡으면 됨
	- 라우팅 구성과 같이 대역이 나뉘어져 있는 경우에만 pask의 인터페이스로 (이중화인 경우 이중화 vip로) 설정
```
switch# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      : 192.168.193.1

    Network
       Destination      Gateway Interface Priority HC-ID HC-Type HC-Result Description
       192.168.193.0/24 0.0.0.0 lan
================================================================================
```
- vlan
```
switch# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                                      | xg  |          |         |
          |         |                   1 1 1 1 1 1 1 1 1 1 2 |     |          |         |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 | 1 2 | TBM      | ND      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u u . u u u u u u u u u u | u u | disable  | 1       |
lan       | 10      | . . . . . . . . . u . . . . . . . . . . | . . | disable  | 1       |
================================================================================
```

## vrrp 설정
- vrrp 설정
	- 인터페이스 vip는 서로 통신만 가능하면 되기 때문에 아무거나 잡아도 된다
```
slb2(config-failover)# show vrrp 1

================================================================================
  VRRP: 1
 ------------------------------------------------------------------------------
    VRID                          : 1
    Ndomain                       : 1
    Mode                          : active-standby
    Status                        : enable
    Priority                      : 101
    Service VIP                   :
    Send GARP All SVIP            : disable
    Port boundary                 :
    Port-Block                    :

    Interface
       VLAN  Advertise VIP       VMAC              Overlapped
       lan   enable    2.2.2.250 00:00:5e:00:01:01
       wan   enable    1.1.1.250 00:00:5e:00:01:01

    Advertise Interval (100 msec) : 10
    Retry                         : 3
    Arp Count                     : 0
    VMAC                          : enable
    Preemption                    : enable
    Description                   :

    Track
        Member port               :

        Single Port
           Priority Port
           1        ge11

        Track real                :
================================================================================



SLB1(config-failover)# show vrrp 1

================================================================================
  VRRP: 1
 ------------------------------------------------------------------------------
    VRID                          : 1
    Ndomain                       : 1
    Mode                          : active-standby
    Status                        : enable
    Priority                      : 101
    Service VIP                   :
    Send GARP All SVIP            : disable
    Port boundary                 :
    Port-Block                    :

    Interface
       VLAN  Advertise VIP       VMAC              Overlapped
       lan   enable    2.2.2.250 00:00:5e:00:01:01
       wan   enable    1.1.1.250 00:00:5e:00:01:01

    Advertise Interval (100 msec) : 10
    Retry                         : 3
    Arp Count                     : 0
    VMAC                          : enable
    Preemption                    : enable
    Description                   :

    Track
        Member port               :

        Single Port
           Priority Port
           1        ge11

        Track real                :
================================================================================
```
- 상태
```
slb2(config)# show failover

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
       1     active-standby master  enable 102   101      lan   2.2.2.250 00:00:5e:00:01:01
                                                          wan   1.1.1.250 00:00:5e:00:01:01
```

## 동작 확인
- entry 확인
```
slb2(config)# show entry
================================================================================
Prot [Org]Sip:Sport        Dip:Dport          - [Rep]Sip:Sport       Dip:Dport             Svc:Real   [R]Svc:Real  ND
--------------------------------------------------------------------------------
tcp  192.168.212.182:4672  192.168.193.130:80 - 192.168.193.132:8080 192.168.212.182:4672  slb.test:4              1
tcp  192.168.212.182:50101 192.168.193.130:80 - 192.168.193.132:8080 192.168.212.182:50101 slb.test:4              1
tcp  192.168.212.182:56776 192.168.193.130:80 - 192.168.193.132:8080 192.168.212.182:56776 slb.test:4              1
tcp  192.168.212.182:2258  192.168.193.130:80 - 192.168.193.132:8080 192.168.212.182:2258  slb.test:4              1
tcp  192.168.212.182:45220 192.168.193.130:80 - 192.168.193.132:8080 192.168.212.182:45220 slb.test:4              1
tcp  192.168.212.182:8563  192.168.193.130:80 - 192.168.193.132:8080 192.168.212.182:8563  slb.test:4              1
================================================================================
```

- host routing 통해 lan 인터페이스로 전달
	- `00:00:5e:00:01:01`는 pask의 이중화 vmac
	- 서버는 212.182를 모르니까 gw를 찾는 arp 전송, 그 arp에 대해 pask가 자신의 이중화 vmac으로 응답
```
slb2(config)# tcpdump -nei lan host 192.168.212.182
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
15:16:14.438460 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype IPv4 (0x0800), length 531: 192.168.212.182.4672 > 192.168.193.132.8080: Flags [P.], seq 3362607289:3362607766, ack 2981370027, win 255, length 477
15:16:14.438620 00:06:c4:94:1c:0f > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 60: 192.168.193.132.8080 > 192.168.212.182.4672: Flags [.], ack 477, win 251, length 0
15:16:14.480562 00:06:c4:94:1c:0f > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 1345: 192.168.193.132.8080 > 192.168.212.182.4672: Flags [P.], seq 1:1292, ack 477, win 251, length 1291
15:16:14.503494 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype IPv4 (0x0800), length 392: 192.168.212.182.4672 > 192.168.193.132.8080: Flags [P.], seq 477:815, ack 1292, win 250, length 338
15:16:14.503636 00:06:c4:94:1c:0f > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 60: 192.168.193.132.8080 > 192.168.212.182.4672: Flags [.], ack 815, win 251, length 0
```
- proxy arp 동작 확인
```
switch2(config)# tcpdump -nei lan arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
15:51:13.854760 00:06:c4:94:1c:0f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.193.1 tell 192.168.193.132, length 28
15:51:13.854896 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype ARP (0x0806), length 60: Reply 192.168.193.1 is-at 00:00:5e:00:01:01, length 46
```

```
slb2(config)# tcpdump -nei lan arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
16:02:25.620581 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 2.2.2.250 tell 2.2.2.250, length 28
16:02:27.837165 00:06:c4:94:1b:fb > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.193.1 tell 192.168.193.131, length 46
16:02:27.837186 00:06:c4:94:1c:17 > 00:06:c4:94:1b:fb, ethertype ARP (0x0806), length 42: Reply 192.168.193.1 is-at 00:00:5e:00:01:01, length 28
```

- wan 인터페이스 캡쳐
```
slb2(config)# tcpdump -nei wan host 192.168.212.182
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wan, link-type EN10MB (Ethernet), capture size 65535 bytes
15:38:07.849286 00:d0:f8:22:35:42 > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 531: 192.168.212.182.1406 > 192.168.193.130.80: Flags [P.], seq 62214449:62214926, ack 1045511895, win 255, length 477
15:38:07.891229 00:06:c4:94:1c:17 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 1345: 192.168.193.130.80 > 192.168.212.182.1406: Flags [P.], seq 1:1292, ack 477, win 251, length 1291
```

- 서버 응답
```
switch(config)# tcpdump -nei lan host 192.168.212.182
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes
15:36:49.725782 00:06:c4:94:1c:17 > 00:06:c4:94:1b:fb, ethertype IPv4 (0x0800), length 531: 192.168.212.182.62560 > 192.168.193.131.8080: Flags [P.], seq 3711608615:3711609092, ack 2656841920, win 255, length 477
15:36:49.769410 00:06:c4:94:1b:fb > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 54: 192.168.193.131.8080 > 192.168.212.182.62560: Flags [.], ack 477, win 251, length 0
```

- 이거는 리얼서버의 arp테이블이 지워졌을 때 보내는 것 - 직결이라 물어보는 거 같다 
	- 193.150,152 각각 pask의 lan 인터페이스 ip
	- 특이한 것은 마스터 장비는 vmac으로 답해주고 백업 장비는 실제 mac을 알려줌
```
slb2(config)# tcpdump -nei lan arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lan, link-type EN10MB (Ethernet), capture size 65535 bytes


15:28:25.660986 00:00:5e:00:01:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 2.2.2.250 tell 2.2.2.250, length 28
15:28:30.548796 00:06:c4:94:1c:0f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.193.152 tell 192.168.193.132, length 46
15:28:30.548816 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype ARP (0x0806), length 42: Reply 192.168.193.152 is-at 00:00:5e:00:01:01, length 28
15:28:33.817134 00:06:c4:94:1c:0f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.193.150 tell 192.168.193.132, length 46
15:28:33.817175 00:06:c4:94:1c:17 > 00:06:c4:94:1c:0f, ethertype ARP (0x0806), length 42: Reply 192.168.193.150 is-at 00:06:c4:94:1c:17, length 28
```


# 원암 구성
- 모드 설명
- lan to lan, both nat 주로 사용
	- 차이점: lan to lan은 snat 할 ip 대역을 선택할 수 있음
- 둘 다 반드시 원암일 때만 쓰는 것은 아니고 그냥 snat이 필요할 때 사용
- lan to lan 예시
	- pask에서 서로 다른 서버 그룹에 대해 부하 분산을 하는데 두 그룹이 동일 대역일 때
	  ex. slb1으로 부하 분산 후 해당 slb1가 클라이언트 역할이 되어 slb2로 부하 분산 하는 경우
	  두 그룹이 동일 대역인 경우 l2 통신이 되기 때문에 snat 필요
	  만일 두 그룹이 다른 대역인 경우? ->생각해봐야 함

## pas-k 설정
- 인터페이스
```
SLB1(config)# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address      Broadcast       RPF     ND    Description
  lan     up     00:06:c4:92:02:f3 3.3.3.50/24       3.3.3.255       default 1
  wan     up     00:06:c4:92:02:f3 192.168.193.55/24 192.168.193.255 default 1
  default down   00:06:c4:92:02:f3                                   default 1
  mgmt    up     00:06:c4:92:02:f2 192.168.100.1/24  192.168.100.255 default 0
================================================================================

slb2(config)# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address      Broadcast       RPF     ND    Description
  lan     up     00:06:c4:94:1c:17 3.3.3.52/23       3.3.3.255       default 1
  wan     up     00:06:c4:94:1c:17 192.168.193.56/24 192.168.193.255 default 1
  default down   00:06:c4:94:1c:17                                   default 1
  mgmt    up     00:06:c4:94:1c:16 192.168.100.1/24  192.168.100.255 default 0
================================================================================
```

- vlan (lan 대역 사용X)
	- 구성 상 스위치를 거쳐 라우터로 가는 구성이라 untag로 설정함
```
SLB1(config)# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                                      | xg  |          |         |
          |         |                   1 1 1 1 1 1 1 1 1 1 2 |     |          |         |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 | 1 2 | TBM      | ND      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u u . . u u u u u u u u u | u u | disable  | 1       |
wan       | 10      | . . . . . . . . . u u . . . . . . . . . | . . | disable  | 1       |
lan       | 20      | . . . . . . . . . . . . . . . . . . . . | . . | disable  | 1       |
================================================================================
```

- vrrp(이중화)
```
SLB1(config)# show running-config failover
!
! Application switch configuration (v2.2.7.8.3)
! 2026/07/29 16:09:25
!
! Failover configuration
!
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
    priority 101
    send-garp-all-svip disable
    interface wan vip 192.168.193.135
    interface wan advertise-send enable
    advertise-interval 10
    retry 3
    arp-count 0
    vmac enable
    preemption enable
    track single-port 1 port ge11
    apply
  apply
```

- slb 설정 (lan to lan)
	- lan to lan : snat 수행할 대역 지정 (즉 212 대역으로 들어오는 요청에 한해 snat)
	- snat ip를 여러 개 넣게 되면 설정된 snatip에 대해 돌아가면서 세션을 만든다
		- 도입된 이유: 
		  특정 사이트에서 source port 풀이 매우 좁아서 클라이언트의 source port가 겹치는 사례가 발생하고 있었음
		  (참고: 클라이언트 식별은 sip, sport로 함)
		  그런데 sip랑 sport까지 겹쳐버리는 문제가 발생하여
		  l4에서 snat ip를 여러개 만들어서 전송할 수 있도록 해서 연결이 끊길 확률을 낮추도록 하기 위함
		  snat ip는 반드시 pask에 실제로 존재할 필요 없음, 왜냐하면 어차피 l4가 nat해서 보내주기 때문에 pask mac으로 들어오기 때문
```
slb2(config)# show info slb test

================================================================================
  SLB: test
 ------------------------------------------------------------------------------
    Name                 : test
    IP Version           : ipv4
    Ndomain              : 1
    Status               : enable
    Priority             : 50
    NAT Mode             : lan-to-lan
    LB Method            : rr
    Fail Skip            : none
    Fail Action          : default
    Session Timeout Mode : global
    Session Reset        : none
    Session-sync         : none
    H/C Condition        : all
    Health Check         : 1
    Passive Health Check :
    Service Health       : ACT

    Vip
       VIP             Protocol Vport
       192.168.193.130 tcp      443

    Description          :

    Filter
       ID    Type    Protocol Src IP    Dst IP             Status Hit   Description
       1     include tcp      0.0.0.0/0 192.168.193.130/32 enable 28

    Sticky
        Time             : 60
        Overmax          : enable

        --- Option -----------------------------------------------------------
        Src Subnet       : 255.255.255.255
        Dst Subnet       : 0.0.0.0

    Keep-Backup
        Service          : disable
        Real             : disable

    Backup               :

    Dynamic-Filter       :

    Real
       ID    Name  RIP             Rport Backup Status G-SHDN  Health Cause State-Time      Description
       5     ticon 192.168.181.153 8443         enable disable ACT          0 days 07:04:51
       6           192.168.181.91  8443         enable disable ACT          0 days 00:00:24

    Health-Check-Info
       ID    Type  Port  Status
       1     tcp   8443  enable

    Health-Check-Result
        ID Name  Total Active: 1
        5  ticon     O         O (1 ms)
        6            O         O (1 ms)

    Real Health-Check-Result
        ID Total
        5      D
        6      D

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           5     ticon 1                28
           6           0                0

        CPS              : 0
        Current sessions : 1
        Total sessions   : 245

    Service-Chain-Info   :


    --- Option ---------------------------------------------------------------
    Src NAT IP           : 192.168.193.130
    Lan to Lan           : 192.168.212.0/24
================================================================================
```

- real 설정
```
slb2(config)# show real

================================================================================
REAL Configuration
 ------------------------------------------------------------------------------
  ID    Name  RIP             Rport SSL-Rport Weight Backup SVC-IP Status Description
  5     ticon 192.168.181.153 8443            1                    enable
  6           192.168.181.91  8443            1                    enable
================================================================================
```

### 동작 확인
- entry 확인
	- pask에서 192.168.193.130으로 snat, rip로 dnat 해서 리얼서버로 전송
	- 서버에서는 gw를 통해 라우터로 가지 않고 pask mac으로 전송하게 된다
		-  그 후 pask에서 다시 
```
slb2# show entry
================================================================================
Prot [Org]Sip:Sport        Dip:Dport           - [Rep]Sip:Sport       Dip:Dport             Svc:Real   [R]Svc:Real  ND
--------------------------------------------------------------------------------
tcp  192.168.212.182:9749  192.168.193.130:443 - 192.168.181.153:8443 192.168.193.130:9749  slb.test:5              1
tcp  192.168.212.182:48391 192.168.193.130:443 - 192.168.181.153:8443 192.168.193.130:48391 slb.test:5              1
tcp  192.168.212.182:51126 192.168.193.130:443 - 192.168.181.153:8443 192.168.193.130:51126 slb.test:5              1
tcp  192.168.212.182:10987 192.168.193.130:443 - 192.168.181.153:8443 192.168.193.130:10987 slb.test:5              1
tcp  192.168.212.182:45398 192.168.193.130:443 - 192.168.181.153:8443 192.168.193.130:45398 slb.test:5              1

생략
```

- 덤프
	- 맨 처음 vip로 요청은 vmac으로 (이중화)
```
slb2# tcpdump -nei wan host 192.168.193.130
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wan, link-type EN10MB (Ethernet), capture size 65535 bytes
17:28:08.123849 00:d0:f8:22:35:42 > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 901: 192.168.212.182.36452 > 192.168.193.130.443: Flags [P.], seq 386852068:386852915, ack 3655516829, win 255, length 847
17:28:08.123886 00:06:c4:94:1c:17 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 901: 192.168.193.130.36452 > 192.168.181.153.8443: Flags [P.], seq 386852068:386852915, ack 3655516829, win 255, length 847
17:28:08.139122 00:06:c4:62:ae:7e > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 536: 192.168.181.153.8443 > 192.168.193.130.36452: Flags [P.], seq 1:483, ack 847, win 501, length 482
17:28:08.139135 00:06:c4:94:1c:17 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 536: 192.168.193.130.443 > 192.168.212.182.36452: Flags [P.], seq 1:483, ack 847, win 501, length 482
```

- 서버
	- `01:b5` 181대역 gw mac
```
[root@localhost ~]# tcpdump -nnei ens3 host 192.168.193.130
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:25:55.112033 00:06:c4:62:ae:7e > fa:16:3e:47:0d:49, ethertype IPv4 (0x0800), length 66: 192.168.193.130.39023 > 192.168.181.153.8443: Flags [S], seq 3407470341, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0
11:25:55.112134 fa:16:3e:47:0d:49 > 00:00:5e:00:01:b5, ethertype IPv4 (0x0800), length 66: 192.168.181.153.8443 > 192.168.193.130.39023: Flags [S.], seq 678771786, ack 3407470342, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
11:25:55.112896 00:06:c4:62:ae:7e > fa:16:3e:47:0d:49, ethertype IPv4 (0x0800), length 54: 192.168.193.130.39023 > 192.168.181.153.8443: Flags [.], ack 1, win 255, length 0
11:25:55.113844 00:06:c4:62:ae:7e > fa:16:3e:47:0d:49, ethertype IPv4 (0x0800), length 1793: 192.168.193.130.39023 > 192.168.181.153.8443: Flags [P.], seq 1:1740, ack 1, win 255, length 1739
11:25:55.113862 fa:16:3e:47:0d:49 > 00:00:5e:00:01:b5, ethertype IPv4 (0x0800), length 54: 192.168.181.153.8443 > 192.168.193.130.39023: Flags [.], ack 1740, win 495, length 0
11:25:55.114570 fa:16:3e:47:0d:49 > 00:00:5e:00:01:b5, ethertype IPv4 (0x0800), length 220: 192.168.181.153.8443 > 192.168.193.130.39023: Flags [P.], seq 1:167, ack 1740, win 501, length 166
11:25:55.114710 00:06:c4:62:ae:7e > fa:16:3e:47:0d:49, ethertype IPv4 (0x0800), length 66: 192.168.193.130.3950 > 192.168.181.153.8443: Flags [S], seq 502671070, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0
```
## 오류
### http 접속
- l4 장비는 프로토콜(http,https)까지는 변경 불가함
	- l7 기능 사용 필요 (xforwarded port 등)
- 리얼서버가 https로 접속해야 하는 경우 애초에 vprot를 443으로 잡는 것이 좋음
- L7 동작은 pask-클라이언트, pask-서버 이렇게 세션을 두 개 맺어서 관리함
	- L4 동작은 세션을 하나만 맺음

### pas-k가 RST을 보내는 경우
- **mpi로 처리되었기 때문**
- ex) vrrp vip를 snatip로 설정함, 서버에서 vrrp vip ip로 요청이 들어가기 때문에 mpi로 처리되었음
	- 192.168.193.135 -> vrrp vip
	- 현재 클라이언트 요청 포트는 `42838` 이다. 따라서 mpi에는 해당 포트가 열려있지 않기 때문에 reset
```
SLB1# tcpdump -nei wan host 192.168.193.135
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wan, link-type EN10MB (Ethernet), capture size 65535 bytes
09:49:56.477739 00:06:c4:92:02:f3 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 66: 192.168.193.135.42838 > 192.168.181.153.8443: Flags [S], seq 309259093, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0
09:49:56.478204 00:06:c4:62:ae:7e > 00:00:5e:00:01:01, ethertype IPv4 (0x0800), length 66: 192.168.181.153.8443 > 192.168.193.135.42838: Flags [S.], seq 3409917513, ack 309259094, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
09:49:56.478233 00:06:c4:92:02:f3 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 54: 192.168.193.135.42838 > 192.168.181.153.8443: Flags [R], seq 309259094, win 0, length 0
09:49:57.480874 00:06:c4:92:02:f3 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 66: 192.168.193.135.42838 > 192.168.181.153.8443: Flags [S], seq 309259093, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0
```
- **보통 snatip는 SLB vip로 설정함 (디폴트)**
- 보통 이렇게 (vrrp vip를 snat ip로 설정하는 경우)에는 overlap 설정을 해서 인터페이스 ip로 와도 서비스로 처리되도록 해야 함 

### 와이어샤크 확인 방법
- tcp 3 way handshake는 syn, ack 단계에서는 length가 0
- 3 way handshake 부터 정상적으로 끝났는지 확인 -> 정상적으로 세션이 맺어지고 데이터를 주고 받고 있는지?
	- 서버가 아무것도 보내지 않는 경우 timeout으로 클라이언트가 FIN을 보낼 수도 있다

- https 연결은 tls 연결을 맺어야 함
	- client hello 먼저 전송, 이후 server hello가 와야 함
![[image-2.png|1179]]

- tcp fragment 확인
![[image.png|1386]]
- 489번 총 length 확인
![[image-1.png|1390|1390x219]]
