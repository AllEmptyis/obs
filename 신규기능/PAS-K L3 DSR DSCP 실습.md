# L3 DSR DSCP 설명
- dsr은 클라이언트 응답에 대해 서버가 직접 응답하는 방식
- L4가 서버의 rip로 dnat+dscp 값을 넣어서 주면, 서버는 그 패킷에 대해 vip로 dnat 해서 클라이언트에게 응답하는 방식
- 서버에서 아래와 같은 설정 필요
	- arp 응답 방지 //**L3 구성에서는 필요 없음**
	- lo 인터페이스에 vip/32 설정
	- nftables에서 magle table - input 체인에 특정 dscp 값을 가진 패킷이 들어오면 vip로 daddr 하라는 규칙 추가
		- 이렇게 하면 서버에서 vip를 sip로 해서 응답한다.

# 구성-라우팅
- 테스트는 랩실에서 진행
- 서버: 192.168.211.105
	- gw 192.168.211.1로 잡으면 됨
- PAS-K: 192.168.193.100
	- gw 192.168.193.1로 잡아준다

# L4 설정
## PAS-K 인터페이스
- vlan
```
test# show vlan

================================================================================
VLAN Configuration
--------------------------------------------------------------------------------
          |         | ge                              |          |
          |         |                   1 1 1 1 1 1 1 |          |
VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 | TBM      | Description
--------------------------------------------------------------------------------
default   | 1       | u u u u u u u u . u u u u u u u | disable  |
test      | 10      | . . . . . . . . u . . . . . . . | disable  |
================================================================================
```
- 라우팅
```
test# show route

================================================================================
  ROUTE
 ------------------------------------------------------------------------------
    Default-Gateway      :

    Network
       Destination      Gateway       Interface HC-ID HC-Type HC-Result Description
       192.168.193.0/24 0.0.0.0       test
       192.168.211.0/24 192.168.193.1 test
       192.168.212.0/24 192.168.193.1 test
================================================================================
```
## real server 구성
- 인터페이스 잡아주는 것 주의 (안잡으면 헬스체크 안됨)
```
test# show real 1

================================================================================
  REAL: 1
 ------------------------------------------------------------------------------
    ID                       : 1
    Name                     : test
    RIP                      : 192.168.211.105
    Rport                    : 8080
    SSL Rport                :
    Priority                 : 0
    Weight                   : 1
    Interface                : test
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
## health check (dscp icmp)
```
test# show health-check 1

================================================================================
  HEALTH-CHECK: 1
 ------------------------------------------------------------------------------
    ID                   : 1
    Type                 : dscp-icmp
    Timeout              : 3
    Interval             : 5
    Retry                : 3
    Recover              : 0
    Status               : enable
    Graceful Shutdown    : disable
    Description          :

    --- Option ---------------------------------------------------------------
    SIP                  :
    TIP                  : 192.168.193.105
    DSCP                 : 10
================================================================================
```
## slb
```
test# show info slb  test

================================================================================
  SLB: test
 ------------------------------------------------------------------------------
    Name                 : test
    IP Version           : ipv4
    Status               : enable
    Priority             : 50
    NAT Mode             : l3dsr-dscp
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
       VIP             DSCP  Protocol Vport
       192.168.193.105 10    all      8080

    Description          :

    Filter
       ID    Type    Protocol Src IP    Dst IP             Status Hit   Description
       3     include all      0.0.0.0/0 192.168.193.105/32 enable 658

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
       ID    Name  RIP             Rport Backup Status G-SHDN  Health Cause State-Time      Description
       1     test  192.168.211.105 8080         enable disable ACT          0 days 00:20:40

    Health-Check-Info
       ID    Type      Port  Status
       1     dscp-icmp 0     enable

    Health-Check-Result
        ID Name Total Active: 1
        1  test     O         O (156 ms)

    Real Health-Check-Result
        ID Name Total 1
        1  test     O O (153 ms)

    Statistics
        Real
           ID    Name  Current sessions Total sessions
           1     test  0                204

        Current sessions : 0
        Total sessions   : 204

================================================================================
```

# 서버 설정
## 인터페이스
```
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 192.168.193.105/32 scope global lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether d4:ae:52:ca:a2:98 brd ff:ff:ff:ff:ff:ff
    altname enp2s0f0
3: eno2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether d4:ae:52:ca:a2:99 brd ff:ff:ff:ff:ff:ff
    altname enp2s0f1
    inet 192.168.211.105/24 brd 192.168.211.255 scope global noprefixroute eno2
       valid_lft forever preferred_lft forever
    inet6 fe80::d6ae:52ff:feca:a299/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```
- nginx
```
[root@localhost ~]# ss -lntup | grep 8080
tcp   LISTEN 0      511          0.0.0.0:8080       0.0.0.0:*    users:(("nginx",pid=215889,fd=6),("nginx",pid=215888,fd=6),("nginx",pid=215887,fd=6),("nginx",pid=215886,fd=6),("nginx",pid=215885,fd=6),("nginx",pid=215884,fd=6),("nginx",pid=215883,fd=6),("nginx",pid=215882,fd=6),("nginx",pid=215881,fd=6))
tcp   LISTEN 0      511             [::]:8080          [::]:*    users:(("nginx",pid=215889,fd=7),("nginx",pid=215888,fd=7),("nginx",pid=215887,fd=7),("nginx",pid=215886,fd=7),("nginx",pid=215885,fd=7),("nginx",pid=215884,fd=7),("nginx",pid=215883,fd=7),("nginx",pid=215882,fd=7),("nginx",pid=215881,fd=7))
```
## nf table 설정
- 주의: nat 테이블에서 prerouting으로 하면 동작 안함 (보낼 때 snat 안해서 서버에서 리셋 보냄)
```
[root@localhost ~]# nft list ruleset
table ip mangle {
        chain INPUT {
                type filter hook input priority mangle; policy accept;
                ip dscp 0x0a counter packets 880 bytes 47326
                ip dscp af11 ip daddr set 192.168.193.105
        }

        chain PREROUTING {
                type filter hook prerouting priority mangle; policy accept;
        }

        chain OUTPUT {
                type filter hook output priority mangle; policy accept;
        }
}
table ip nat {
        chain PREROUTING {
                type nat hook prerouting priority dstnat; policy accept;
        }

        chain POSTROUTING {
                type nat hook postrouting priority srcnat; policy accept;
        }
}
```

# 결과 - slb
```
[root@localhost ~]# tcpdump -nei eno2 host 192.168.212.163 and not port 22
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
13:51:31.582945 00:06:c4:62:ae:7e > d4:ae:52:ca:a2:99, ethertype IPv4 (0x0800), length 618: 192.168.212.163.64031 > 192.168.211.105.webcache: Flags [P.], seq 2657193698:2657194262, ack 887858812, win 255, length 564: HTTP: GET / HTTP/1.1
13:51:31.583163 d4:ae:52:ca:a2:99 > 00:00:5e:00:01:d3, ethertype IPv4 (0x0800), length 235: 192.168.193.105.webcache > 192.168.212.163.64031: Flags [P.], seq 887858812:887858993, ack 2657194262, win 501, length 181: HTTP: HTTP/1.1 304 Not Modified
13:51:31.629253 00:06:c4:62:ae:7e > d4:ae:52:ca:a2:99, ethertype IPv4 (0x0800), length 60: 192.168.212.163.64031 > 192.168.211.105.webcache: Flags [.], ack 182, win 254, length 0
13:51:34.755075 00:06:c4:62:ae:7e > d4:ae:52:ca:a2:99, ethertype IPv4 (0x0800), length 618: 192.168.212.163.64031 > 192.168.211.105.webcache: Flags [P.], seq 564:1128, ack 182, win 254, length 564: HTTP: GET / HTTP/1.1
13:51:34.755211 d4:ae:52:ca:a2:99 > 00:00:5e:00:01:d3, ethertype IPv4 (0x0800), length 235: 192.168.193.105.webcache > 192.168.212.163.64031: Flags [P.], seq 181:362, ack 565, win 501, length 181: HTTP: HTTP/1.1 304 Not Modified
13:51:34.805783 00:06:c4:62:ae:7e > d4:ae:52:ca:a2:99, ethertype IPv4 (0x0800), length 60: 192.168.212.163.64031 > 192.168.211.105.webcache: Flags [.], ack 363, win 253, length 0
13:51:35.642278 00:06:c4:62:ae:7e > d4:ae:52:ca:a2:99, ethertype IPv4 (0x0800), length 618: 192.168.212.163.64031 > 192.168.211.105.webcache: Flags [P.], seq 1128:1692, ack 363, win 253, length 564: HTTP: GET / HTTP/1.1
```

# 결과 - health check
- dscp 10 설정된 값으로 헬스체크 응답 확인
	- 원리: dscp 10 넣어서 rip로 dnat 해서 전송하면 서버에서 vip로 nat 되어서 오는지 체크
```
test# tcpdump -nei test icmp -vv
tcpdump: listening on test, link-type EN10MB (Ethernet), capture size 65535 bytes
16:12:47.354333 00:06:c4:84:10:07 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 42: (tos 0x28, ttl 64, id 1, offset 0, flags [none], proto ICMP (1), length 28)
    192.168.193.100 > 192.168.211.105: ICMP echo request, id 62143, seq 0, length 8
16:12:47.354461 00:06:c4:84:10:07 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 42: (tos 0x28, ttl 64, id 1, offset 0, flags [none], proto ICMP (1), length 28)
    192.168.193.100 > 192.168.211.105: ICMP echo request, id 41330, seq 0, length 8
16:12:47.354560 00:06:c4:62:ae:7e > 00:06:c4:84:10:07, ethertype IPv4 (0x0800), length 60: (tos 0x28, ttl 63, id 42155, offset 0, flags [none], proto ICMP (1), length 28)
    192.168.193.105 > 192.168.193.100: ICMP echo reply, id 62143, seq 0, length 8
16:12:47.354659 00:06:c4:62:ae:7e > 00:06:c4:84:10:07, ethertype IPv4 (0x0800), length 60: (tos 0x28, ttl 63, id 42156, offset 0, flags [none], proto ICMP (1), length 28)
    192.168.193.105 > 192.168.193.100: ICMP echo reply, id 41330, seq 0, length 8
```