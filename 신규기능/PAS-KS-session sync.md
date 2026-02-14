# session sync
- fail over 구성에서 마스터-백업 장비가 전환될 때 세션 유지를 위해 주기적으로 세션싱크를 주고받는 기능
- 몇몇 클라우드 환경에서 기존 loopback ip를 사용하여 전송 시 drop 하는 현상 확인
- 지정한 인터페이스 ip로 ss 패킷을 주고받을 수 있도록 기능 개발
	- live update ip 항목 추가
- v2.2.7.2.0 이상
# 구성 과정
- pas-ks 2대, real server vm 1대, 
- slb 생성
	- dnat로 구성
- real server (vm)
	- gateway를 ks의 vrrp vip로 잡아야 한다 (클라이언트 sip로 응답할 때 디폴트 게이트웨이를 찾아야 하기 때문)
	- nginx 설치 및 8080 포트 변경, 리스닝 확인
	- iptables, firewalld stop
- failover 생성 (vrrp)
	- session sync 구성
# slb
```
test5# show slb test

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
    Session-sync         : all
    H/C Condition        : all
    Health Check         : 1
    Passive Health Check :

    Vip
       VIP         Protocol Vport
       10.10.10.10 tcp      80

    Description          :

    Sticky
        Time             : 60

        --- Option -----------------------------------------------------------
        Src Subnet       : 255.255.255.255
        Dst Subnet       : 0.0.0.0

    Keep-Backup
        Service          : disable
        Real             : disable

    Backup               :

    Dynamic-Filter       :

    Real
       ID    Rport Backup Status Description
       1     8080         enable
================================================================================
```
- real
```
test5# show real

================================================================================
REAL Configuration
 ------------------------------------------------------------------------------
  ID    Name  RIP      Rport SSL-Rport Weight Backup SVC-IP Status Description
  1     real1 5.5.5.10 8080            1                    enable
================================================================================
```
# failover
```
test5(config)# show failover

================================================================================
  FAILOVER
 ------------------------------------------------------------------------------
    Delay-Time             : 10

    Session-Sync
        Status                     : enable
        Live update interval (10 msec) : 100
        Full update interval (sec) : 10
        Update method              : full
        Live Update IP address     : interface-ip
        Peer                       : node2

        Interface
            Name                   : ss
            IPv4 Address           : 11.11.11.5/24
            Peer IP Address        : 11.11.11.6
            Hc-Retry               : 3
            Health                 : act

    A-A Failover
        Method             : disable

    Vrrp
       VRID  Mode           Running Status Total Priority Interface VIP      VMAC  Description
       1     active-standby master  enable 101   100      lan       5.5.5.50

    Vrrp6                  :

    Ha
        Running                   : stop
        Status                    : disable

        Interface                 :

        Node                      :

        Default State             : master
        Heartbeat Interval (100 msec) : 10
        Retry                     : 3
        VMAC                      : enable

    SSL-session-cache-sync
        Status           : disable

        Interface
            Name         :
            IPv4 Address :
            Peer IP Address :
================================================================================
```
# entry 확인
```
test5# show entry
================================================================================
Prot [Org]Sip:Sport   Dip:Dport      - [Rep]Sip:Sport Dip:Dport        Svc:Real   [R]Svc:Real
--------------------------------------------------------------------------------
tcp  10.10.10.1:59194 10.10.10.10:80 - 5.5.5.10:8080  10.10.10.1:59194 slb.test:1
tcp  10.10.10.1:61286 10.10.10.10:80 - 5.5.5.10:8080  10.10.10.1:61286 slb.test:1
================================================================================
```
- conntrack -L
```
root@test5:~# conntrack -L
conntrack v1.4.6 (conntrack-tools): 0 flow entries have been shown.
root@test5:~# conntrack -L
tcp      6 3598 ESTABLISHED src=10.10.10.1 dst=10.10.10.10 sport=52834 dport=80 src=5.5.5.10 dst=10.10.10.1 sport=8080 dport=52834 [ASSURED] mark=0 use=1 Service=slb.test Dest=1 Property=P TunnelType=none TunnelVni=0 TunnelUptime=6496
tcp      6 3598 ESTABLISHED src=10.10.10.1 dst=10.10.10.10 sport=51001 dport=80 src=5.5.5.10 dst=10.10.10.1 sport=8080 dport=51001 [ASSURED] mark=0 use=1 Service=slb.test Dest=1 Property=P TunnelType=none TunnelVni=0 TunnelUptime=6496
conntrack v1.4.6 (conntrack-tools): 2 flow entries have been shown.
```
# session sync 패킷 확인
- 3780은 세션싱크 헬스 패킷
```
test5# tcpdump -i ss -n -vvv -X -s 0 "host 11.11.11.5 and port 3781"
tcpdump: listening on ss, link-type EN10MB (Ethernet), capture size 65535 bytes
15:47:22.274882 IP (tos 0x0, ttl 64, id 0, offset 0, flags [none], proto UDP (17), length 148)
    11.11.11.5.43520 > 11.11.11.6.3781: [udp sum ok] UDP, length 120
        0x0000:  4500 0094 0000 0000 4011 4e39 0b0b 0b05  E.......@.N9....
        0x0010:  0b0b 0b06 aa00 0ec5 0080 7697 0002 7800  ..........v...x.
        0x0020:  0001 0106 d7b3 0050 1f90 0000 d7b3 0a0a  .......P........
        0x0030:  0a01 0a0a 0a0a 0505 050a 0000 0000 0a0a  ................
        0x0040:  0a01 0000 0000 0000 0000 0000 0000 0000  ................
        0x0050:  0000 0000 0000 0000 0000 0001 0106 d6e2  ................
        0x0060:  0050 1f90 0000 d6e2 0a0a 0a01 0a0a 0a0a  .P..............
        0x0070:  0505 050a 0000 0000 0a0a 0a01 0000 0000  ................
        0x0080:  0000 0000 0000 0000 0000 0000 0000 0000  ................
        0x0090:  0000 0000    
        

15:43:32.595411 08:00:27:70:e9:c8 > 08:00:27:6c:57:f0, ethertype IPv4 (0x0800), length 220: 11.11.11.5.43520 > 11.11.11.6.3781: UDP, length 178
15:43:32.892256 08:00:27:6c:57:f0 > 08:00:27:70:e9:c8, ethertype IPv4 (0x0800), length 60: 11.11.11.6.58768 > 11.11.11.5.3780: UDP, length 16
15:43:32.930033 08:00:27:70:e9:c8 > 08:00:27:6c:57:f0, ethertype IPv4 (0x0800), length 58: 11.11.11.5.46388 > 11.11.11.6.3780: UDP, length 16
15:43:33.392571 08:00:27:6c:57:f0 > 08:00:27:70:e9:c8, ethertype IPv4 (0x0800), length 60: 11.11.11.6.58768 > 11.11.11.5.3780: UDP, length 16
15:43:33.430946 08:00:27:70:e9:c8 > 08:00:27:6c:57:f0, ethertype IPv4 (0x0800), length 58: 11.11.11.5.46388 > 11.11.11.6.3780: UDP, length 16
15:43:33.615331 08:00:27:70:e9:c8 > 08:00:27:6c:57:f0, ethertype IPv4 (0x0800), length 104: 11.11.11.5.43520 > 11.11.11.6.3781: UDP, length 62                            ....
```
- 3781 필터링
```
test5(config)# tcpdump -nei ss "host 11.11.11.5 and port 3781"
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ss, link-type EN10MB (Ethernet), capture size 65535 bytes
17:56:02.780979 08:00:27:70:e9:c8 > 08:00:27:6c:57:f0, ethertype IPv4 (0x0800), length 162: 11.11.11.5.43520 > 11.11.11.6.3781: UDP, length 120
17:56:02.801629 08:00:27:70:e9:c8 > 08:00:27:6c:57:f0, ethertype IPv4 (0x0800), length 104: 11.11.11.5.43520 > 11.11.11.6.3781: UDP, length 62
17:56:04.390865 08:00:27:70:e9:c8 > 08:00:27:6c:57:f0, ethertype IPv4 (0x0800), length 104: 11.11.11.5.43520 > 11.11.11.6.3781: UDP, length 62
^CExiting...
Done
```
# slb 통신
```
test5(config)# tcpdump -nei bridge host 10.10.10.1

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on bridge, link-type EN10MB (Ethernet), capture size 65535 bytes
17:49:36.644785 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 166: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 238800023:238800135, ack 1842670281, win 165, length 112
17:49:36.645275 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 390: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 112:448, ack 1, win 165, length 336
17:49:36.645846 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 134: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 448:528, ack 1, win 165, length 80
17:49:36.646046 8c:b0:e9:50:e0:c2 > 08:00:27:ba:89:17, ethertype IPv4 (0x0800), length 60: 10.10.10.1.53187 > 10.10.10.5.22: Flags [.], ack 448, win 251, length 0
17:49:36.646269 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 134: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 528:608, ack 1, win 165, length 80
17:49:36.646611 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 150: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 608:704, ack 1, win 165, length 96
17:49:36.646718 8c:b0:e9:50:e0:c2 > 08:00:27:ba:89:17, ethertype IPv4 (0x0800), length 60: 10.10.10.1.53187 > 10.10.10.5.22: Flags [.], ack 608, win 251, length 0
17:49:36.647113 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 118: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 704:768, ack 1, win 165, length 64
17:49:36.647225 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 134: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 768:848, ack 1, win 165, length 80
17:49:36.647583 8c:b0:e9:50:e0:c2 > 08:00:27:ba:89:17, ethertype IPv4 (0x0800), length 60: 10.10.10.1.53187 > 10.10.10.5.22: Flags [.], ack 768, win 250, length 0
17:49:36.647611 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 262: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 848:1056, ack 1, win 165, length 208
17:49:36.647842 8c:b0:e9:50:e0:c2 > 08:00:27:ba:89:17, ethertype IPv4 (0x0800), length 60: 10.10.10.1.53187 > 10.10.10.5.22: Flags [.], ack 1056, win 255, length 0
17:49:36.647980 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 118: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 1056:1120, ack 1, win 165, length 64
17:49:36.648158 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 134: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 1120:1200, ack 1, win 165, length 80
17:49:36.648333 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 118: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 1200:1264, ack 1, win 165, length 64
17:49:36.648459 8c:b0:e9:50:e0:c2 > 08:00:27:ba:89:17, ethertype IPv4 (0x0800), length 60: 10.10.10.1.53187 > 10.10.10.5.22: Flags [.], ack 1200, win 255, length 0
17:49:36.648730 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 118: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 1264:1328, ack 1, win 165, length 64
17:49:36.648883 08:00:27:ba:89:17 > 8c:b0:e9:50:e0:c2, ethertype IPv4 (0x0800), length 182: 10.10.10.5.22 > 10.10.10.1.53187: Flags [P.], seq 1328:1456, ack 1, win 165, length 128
```