# 참고
- 동일한 필터를 가진 CSLB가 여러 개인 경우
	- 캐시 서버에서 5tuple 모두 동일한 세션이 오게 되면 정상적으로 처리되지 못함
	- 만일 캐시 서버에서 오는 트래픽의 5tuple이 달라졌다면(즉 신규 세션으로 인식) priority대로 lb 서비스에 필터를 매칭해서 처리할 수 있음

- pas-k는 패킷이 들어오면 reply 엔트리를 먼저 보고 처리한다
```
switch(config)# show entry
================================================================================
Prot [Org]Sip:Sport Dip:Dport    - [Rep]Sip:Sport Dip:Dport      Svc:Real    [R]Svc:Real     ND
--------------------------------------------------------------------------------
tcp  2.2.2.50:16000 2.2.2.100:80 - 2.2.2.100:80   2.2.2.50:16000 cslb.test:1                 1
tcp  2.2.2.50:25227 2.2.2.100:80 - 2.2.2.100:80   2.2.2.50:25227 cslb.test:1                 1

2.2.2.100:80 sip, 2.2.2.50 dip를 가진 세션이 들어온다면 해당 엔트리로 인식하고 처리
```

# pas-k에 nat 테이블 넣는 방법
- nat 테이블을 넣어야 하는 이유
	- cslb 포워드 엔트리까지만 생성됨
```
nat를 하지 않으면 캐시서버로부터 동일한 세션이 그대로 pask로 돌아오게 된다
즉 서로 주고 받다가 pask가 다시 클라이언트한테 리셋을 보냄

cslb 리버스 엔트리는 들어온 smac와 real mac을 비교해서 동일한 경우 생성
그러나 먼저 pask는 생성되어 있는 포워드 엔트리와 비교하여 동일한 경우 그 포워드 엔트리 세션이라고 생각함

그래서 캐시 서버에서 5tuple 중 (mac 제외, 생성되어 있는 포워드 엔트리와 비교할 수 있는 것) 한가지라도 바꿔서 다른 세션인 걸로 보내줘야 pask가 그 때 smac-real mac을 비교하게 됨

추가:
이후 slb를 통해 서버를 거쳐서 다시 캐시 서버로 오게 되면 그 때 캐시 서버가 자동으로 nat했던 세션을 원복해준다
(리눅스 내부의 conntrack을 통해 관리함)
세션을 원복해줘야 기존 클라이언트와 통신할 수 있음
```
- 명령어
	- shell에서 `ns-svc` 진입
	- iptables -t nat -A POSTROUTING -p tcp -j SNAT --to-source :600-700
```
root@cache(svc_net):~# iptables -t nat -L
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DNAT       all  -- !br0                  anywhere             connmark match  0x726 to:203.0.113.100

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  anywhere            !br0                  connmark match  0x726
SNAT       tcp  --  anywhere             anywhere             to::600-700
```

- pas-k fast proccessor 해제
	- pask를 서버로 쓸 때는 fast proccessor를 해제해야 한다
		- fast proccessor는 pask에서 속도를 빠르게 하기 위해 패킷을 튜닝하는 방식
			- 해제하지 않으면 pas-k에서 cslb 처리를 정상적으로 못함
	- `echo 0 >/proc/sys/net/ipv4/piolb/fast_process(init_net에서)`

# 설정
## pas-k
- 포트 바운더리, 라우팅은 생략
- 인터페이스
```
switch# show int

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address     Broadcast       RPF     ND    Description
  test    up     00:06:c4:94:1c:0f 2.2.2.1/24       2.2.2.255       default 1
  default down   00:06:c4:94:1c:0f                                  default 1
  mgmt    up     00:06:c4:94:1c:0e 192.168.100.1/24 192.168.100.255 default 0
================================================================================
```
- real
```
switch# show real

================================================================================
REAL Configuration
 ------------------------------------------------------------------------------
  ID    Name  RIP       Rport SSL-Rport Weight Backup SVC-IP Status Description
  1     cs    2.2.2.11                  1                    enable
  2     slb_1 2.2.2.101 8080            1                    enable
  3     slb_2 2.2.2.102 8080            1                    enable
================================================================================
```

- H/C
	- cslb에 대한 h/c는 단순 icmp 동작
```
switch# show health-check

================================================================================
HEALTH-CHECK Configuration
 ------------------------------------------------------------------------------
  ID    Type  Timeout Interval Status Port  Description
  1     icmp  3       5        enable 0
  2     tcp   3       5        enable 8080
================================================================================

switch# show health-check 2

================================================================================
  HEALTH-CHECK: 2
 ------------------------------------------------------------------------------
    ID                   : 2
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

- SLB
```
switch# show info slb test

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
    Health Check         : 2
    Passive Health Check :
    Service Health       : ACT

    Vip
       VIP       Protocol Vport
       2.2.2.100 tcp      80

    Description          :

    Filter
       ID    Type    Protocol Src IP    Dst IP       Status Hit   Description
       1     include tcp      0.0.0.0/0 2.2.2.100/32 enable 48

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
       2     slb_1 2.2.2.101 8080         enable disable ACT          0 days 02:37:59
       3     slb_2 2.2.2.102 8080         enable disable ACT          0 days 02:37:59

    Health-Check-Info
       ID    Type  Port  Status
       2     tcp   8080  enable

    Health-Check-Result
        ID Name  Total Active: 2
        2  slb_1     O         O (1 ms)
        3  slb_2     O         O (0 ms)

    Real Health-Check-Result
        ID Total
        2      D
        3      D

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           2     slb_1 0                16
           3     slb_2 0                32

        CPS              : 0
        Current sessions : 0
        Total sessions   : 48

    Service-Chain-Info   :

================================================================================
```

- CSLB
	- 실무에서 filter는 그냥 any any (0.0.0.0/0)로 놓기도 함
```
switch(config)# show info cslb test

================================================================================
  CSLB: test
 ------------------------------------------------------------------------------
    Name                     : test
    IP Version               : ipv4
    Status                   : enable
    Priority                 : 0
    LB Method                : rr
    Fail Skip                : inact
    Reverse Only             : disable
    Recording Cache          : disable
    Transparent Cache        : disable
    Reset reuse connection   : disable
    Graceful Reverse Service : enable
    Session Timeout Mode     : global
    Session Reset            : none
    Session-sync             : none
    H/C Condition            : all
    Health Check             : 1
    Passive Health Check     :
    Service Health           : ACT
    Description              :

    Filter
       ID    Type    Protocol Src IP       Dst IP       Status Hit   Description
       1     include all      2.2.2.100/24 0.0.0.0/0    enable 2
       2     include all      0.0.0.0/0    2.2.2.100/24 enable 2

    Sticky
        Time                 : 60
        Overmax              : enable

        --- Option -----------------------------------------------------------
        Src subnet           : 255.255.255.255
        Dst subnet           : 255.255.255.255

    Keep-Backup
        Service              : disable
        Real                 : disable

    Backup                   :

    Dynamic-Filter           :

    Real
       ID    Name  RIP      Rport Backup Status G-SHDN  Health Cause State-Time      Description
       1     cs    2.2.2.11              enable disable ACT          0 days 05:15:48

    Health-Check-Info
       ID    Type  Port  Status
       1     icmp  0     enable

    Health-Check-Result
        ID Name Total Active: 1
        1  cs       O         O (0 ms)

    Real Health-Check-Result
        ID Total
        1      D

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           1     cs    4                1812

        CPS                  : 0
        Current sessions     : 4
        Total sessions       : 1812

    Service-Chain-Info       :

================================================================================
```

## 동작 확인
- entry 확인
```
switch(config)# show entry
================================================================================
Prot [Org]Sip:Sport Dip:Dport    - [Rep]Sip:Sport Dip:Dport      Svc:Real    [R]Svc:Real     ND
--------------------------------------------------------------------------------
tcp  2.2.2.50:16000 2.2.2.100:80 - 2.2.2.100:80   2.2.2.50:16000 cslb.test:1                 1
tcp  2.2.2.50:25227 2.2.2.100:80 - 2.2.2.100:80   2.2.2.50:25227 cslb.test:1                 1
tcp  2.2.2.50:679   2.2.2.100:80 - 2.2.2.101:8080 2.2.2.50:679   slb.test:2  [R]cslb.test:1  1
tcp  2.2.2.50:656   2.2.2.100:80 - 2.2.2.101:8080 2.2.2.50:656   slb.test:2  [R]cslb.test:1  1
================================================================================
```

- 덤프 확인
	- 1c:0f : pask
	- 1c:03: 캐시 서버
	- 
```
switch(config)# tcpdump -nei test host 2.2.2.100
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on test, link-type EN10MB (Ethernet), capture size 65535 bytes
18:26:38.832972 8c:b0:e9:50:e0:c2 > 00:06:c4:94:1c:0f, ethertype IPv4 (0x0800), length 66: 2.2.2.50.51965 > 2.2.2.100.80: Flags [S], seq 2304988182, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0

18:26:38.833022 00:06:c4:94:1c:0f > 00:06:c4:94:1c:03, ethertype IPv4 (0x0800), length 66: 2.2.2.50.51965 > 2.2.2.100.80: Flags [S], seq 2304988182, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0

18:26:38.833102 00:06:c4:94:1c:03 > 00:06:c4:94:1c:0f, ethertype IPv4 (0x0800), length 66: 2.2.2.50.654 > 2.2.2.100.80: Flags [S], seq 2304988182, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0

18:26:38.833189 8c:b0:e9:50:e0:c2 > 00:06:c4:94:1c:0f, ethertype IPv4 (0x0800), length 66: 2.2.2.50.49813 > 2.2.2.100.80: Flags [S], seq 1110747339, win 65535, options [mss 4034,nop,wscale 8,nop,nop,sackOK], length 0


18:26:38.842805 00:06:c4:94:1c:0f > 00:06:c4:94:1c:03, ethertype IPv4 (0x0800), length 66: 2.2.2.100.80 > 2.2.2.50.654: Flags [S.], seq 843590771, ack 2304988183, win 5840, options [mss 1460,nop,nop,sackOK,nop,wscale 8], length 0

18:26:38.842872 00:06:c4:94:1c:03 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 66: 2.2.2.100.80 > 2.2.2.50.51965: Flags [S.], seq 843590771, ack 2304988183, win 5840, options [mss 1460,nop,nop,sackOK,nop,wscale 8], length 0
18:26:38.842878 00:06:c4:94:1c:0f > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 66: 2.2.2.100.80 > 2.2.2.50.51965: Flags [S.], seq 843590771, ack 2304988183, win 5840, options [mss 1460,nop,nop,sackOK,nop,wscale 8], length 0
```