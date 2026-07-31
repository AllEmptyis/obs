# 구성
- SW1의 ge10 포트에 PC 연결
	- pc ip: 2400:a0a0:1015:5002::11
	  mac: 8C-B0-E9-50-E0-C2
	 gw: 2400:a0a0:1015:5002::1
```
sw1
00:06:c4:75:08:bc
TiFRONT% sh vlan
 -------------------------------------------------------------------------------
  PORT            | ge
                  |                   1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2
                  | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8
 -----------------+-----------------------------------------------------------
  SWITCH MODE     | H H H H H H H H H A T H H H H H H H H H H H H H H H H H
 -----------------+-----------------------------------------------------------
  default  (   1) | U U U U U U U U U . T U U U U U U U U U U U U U U U U U
  VLAN0020 (  20) | . . . . . . . . . U t . . . . . . . . . . . . . . . . .
 -------------------------------------------------------------------------------

vlan20                     [up/up]
    2400:a0a0:1015:5002::15


=====
sw2
00:06:c4:75:0a:ca
TiFRONT% sh vlan
 -----------------------------------------------------------------------------------
  PORT            | ge                                                      | agg
                  |                   1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 |
                  | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 | 1
 -----------------+---------------------------------------------------------+-----
  SWITCH MODE     | H H H H H H H H H A T T H H H H H H H H H H H H H H H H | H
 -----------------+---------------------------------------------------------+-----
  default  (   1) | U U U U U U U U U . T T U U U U U U U U U U U U U U U U | U
  VLAN0020 (  20) | . . . . . . . . . U t t . . . . . . . . . . . . . . . . | .
 -----------------------------------------------------------------------------------

vlan1                      [up/up]
    2400:a0a0:1015:5001::1
vlan20                     [up/up]
    2400:a0a0:1015:5002::1

=======
sw3
00:06:c4:52:10:38

TiFRONT% show vlan
 -------------------------------------------------------------------------------
  PORT            | ge
                  |                   1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2
                  | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8
 -----------------+-----------------------------------------------------------
  SWITCH MODE     | H H H H H H H H H H H T H H H H H H H H H H H H H H H H
 -----------------+-----------------------------------------------------------
  default  (   1) | U U U U U U U U U U U T U U U U U U U U U U U U U U U U
  VLAN0020 (  20) | . . . . . . . . . . . t . . . . . . . . . . . . . . . .
 -------------------------------------------------------------------------------

vlan1                      [up/up]
    2400:a0a0:1015:5001::8

TiFRONT% sh ipv6 route
IPv6 Routing Table
Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
       IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, I - IS-IS, B - BGP
Timers: Uptime

S     ::/0 [1/0] via 2400:a0a0:1015:5001::1, vlan1, 00:01:20
C     ::1/128 via ::, loopback, 00:04:22
C     2400:a0a0:1015:5001::/64 via ::, vlan1, 00:02:26
C     fe80::/64 via ::, mon, 00:04:04


TiFRONT% show mac-table
  ageing-time 300
 -----------------------------------------------------------
    No  | VLAN |  PORT  |   MAC ADDRESS  | FWD/DIS | STATIC
 -------+------+--------+----------------+---------+--------
      1 |    1 |    CPU | 0006.c452.1038 | FORWARD | STATIC
      2 |   20 |    CPU | 0006.c452.1038 | FORWARD | STATIC
      3 | 4001 |    CPU | 0006.c452.1038 | FORWARD | STATIC
      4 |    1 |   ge12 | 0006.c475.08bc | FORWARD |
      5 |    1 |   ge12 | 0006.c475.0aca | FORWARD |
      6 |   20 |   ge12 | 0006.c475.0aca | FORWARD |
      7 |   20 |   ge12 | 8cb0.e950.e0c2 | FORWARD |
      8 |   20 |   ge12 | e4c7.6756.a85b | FORWARD |
 -----------------------------------------------------------
```
## 버전 및 설정
- SW1
```
TiFRONT# sh system
---------------------------------------------
system information
---------------------------------------------
Product Name     : TiFRONT V3.1 G2424
Serial number    : R214T7500A02963
BL version       : 7.20
OS version       : 3.1.17
Memory size      : 1024MB
Mgmt MAC address : 00:06:c4:75:08:bc
Power type       : Single
---------------------------------------------
TiFRONT# sh revision
0.5-rc0
TiFRONT#
TiFRONT# sh running-config | inc ipv6
ipv6 enable
 ipv6 address ::1/128
 tunnel mode ipv6ip 6to4
 ipv6 address 2400:a0a0:1015:5002::15/64
```
- SW2
```
TiFRONT# sh system
---------------------------------------------
system information
---------------------------------------------
Product Name     : TiFRONT V3.1 G2424
Serial number    : R214T7500A03226
BL version       : 7.19
OS version       : 3.1.16
Memory size      : 1024MB
Mgmt MAC address : 00:06:c4:75:0a:ca
Power type       : Single
---------------------------------------------
TiFRONT# sh revision
2.13-rc0
TiFRONT# sh running-config | inc ipv6
ipv6 enable
 ipv6 address ::1/128
 tunnel mode ipv6ip 6to4
 ipv6 address 2400:a0a0:1015:5001::1/64
 ipv6 address 2400:a0a0:1015:5002::1/64
```
- SW3
```
TiFRONT% sh system
---------------------------------------------
system information
---------------------------------------------
Product Name     : TiFRONT V3.1 CS2728GP
Serial number    : R217C5200A03921
BL version       : 7.21
OS version       : 3.1.17
Memory size      : 512MB
Mgmt MAC address : 00:06:c4:52:10:38
Power type       : Single
---------------------------------------------
TiFRONT% show revision
1.4-rc0

TiFRONT% show running-config | inc ipv6
ipv6 enable
 ipv6 address ::1/128
 tunnel mode ipv6ip 6to4
 ipv6 address 2400:a0a0:1015:5001::8/64
ipv6 route ::/0 2400:a0a0:1015:5001::1
```
# ipv6 라우팅 테스트
- sw3에 5001::1 로 static routing 잡은 뒤 테스트
	- 라우팅이 존재하는 경우 PC에서 sw3의 vlan ip로 통신 가능
	- 라우팅이 존재하지 않는 경우 PC에서 sw3의 vlan ip로 통신 불가
- SW3에서 mac table dynamic 삭제 후에도 정상 동작
```
TiFRONT# clear mac address-table dynamic

TiFRONT# ping ipv6 2400:a0a0:1015:5002::11
PING 2400:a0a0:1015:5002::11(2400:a0a0:1015:5002::11) 56 data bytes
64 bytes from 2400:a0a0:1015:5002::11: icmp_seq=1 ttl=127 time=3.78 ms
64 bytes from 2400:a0a0:1015:5002::11: icmp_seq=2 ttl=127 time=20.2 ms
```
- PC에서 SW3으로 통신
```
C:\Users\USER>ping 2400:a0a0:1015:5001::8 -t

Ping 2400:a0a0:1015:5001::8 32바이트 데이터 사용:
2400:a0a0:1015:5001::8의 응답: 시간=14ms
2400:a0a0:1015:5001::8의 응답: 시간=3ms
2400:a0a0:1015:5001::8의 응답: 시간=5ms
2400:a0a0:1015:5001::8의 응답: 시간=17ms
2400:a0a0:1015:5001::8의 응답: 시간=5ms
```
- sw3에서 ipv6 routing 삭제 시 통신 안됨
```
TiFRONT(config)% no ipv6 route ::/0 2400:a0a0:1015:5001::1
TiFRONT(config)% ex
TiFRONT% ping ipv6 2400:a0a0:1015:5002::11
connect: Network is unreachable
TiFRONT% conf
Enter configuration commands, one per line.  End with CNTL/Z.
TiFRONT(config)% ipv6 route ::/0 2400:a0a0:1015:5001::1
TiFRONT(config)% ex
TiFRONT% ping ipv6 2400:a0a0:1015:5002::11
PING 2400:a0a0:1015:5002::11(2400:a0a0:1015:5002::11) 56 data bytes
64 bytes from 2400:a0a0:1015:5002::11: icmp_seq=1 ttl=127 time=7.34 ms
64 bytes from 2400:a0a0:1015:5002::11: icmp_seq=2 ttl=127 time=17.5 ms

--- 2400:a0a0:1015:5002::11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1009ms
rtt min/avg/max/mdev = 7.346/12.471/17.597/5.126 ms
```
# mgmt 포트 ping 테스트
- mgmt0 인터페이스에 ip 할당
	- 통신 가능 (CS2728GP, G2424 확인)
	- 아래 결과는 G2424
```
TiFRONT% sh ipv6 int b
gre0                       [administratively down/down]
    unassigned
ip6tnl0                    [administratively down/down]
    unassigned
loopback                   [up/up]
    ::1
mgmt0                      [up/down]
    2400:a0a0:1015:5002::15
mgmtcs                     [up/up]
    fe80::206:c4ff:fe75:8bc
mon                        [up/up]
    fe80::206:c4ff:fe75:8bd
sit0                       [administratively down/down]
    unassigned
smon                       [up/up]
    fe80::206:c4ff:fe75:8bd
tunl0                      [administratively down/down]
    unassigned
vlan1                      [up/up]
    fe80::206:c4ff:fe75:8bc
vlan20                     [up/up]
    fe80::206:c4ff:fe75:8bc
    
    
    
결과
C:\Users\USER>ping 2400:a0a0:1015:5002::15 -t

Ping 2400:a0a0:1015:5002::15 32바이트 데이터 사용:
2400:a0a0:1015:5002::15의 응답: 시간=1ms
2400:a0a0:1015:5002::15의 응답: 시간<1ms
2400:a0a0:1015:5002::15의 응답: 시간<1ms
```
- G2424에 라우팅 설정 들어가 있음
```
TiFRONT% show ipv6 route
IPv6 Routing Table
Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
       IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, I - IS-IS, B - BGP
Timers: Uptime

S     ::/0 [1/0] via 2400:a0a0:1015:5002::1, mgmt0, 00:19:47
C     ::1/128 via ::, loopback, 18:44:41
C     2400:a0a0:1015:5002::/64 via ::, mgmt0, 00:23:42
C     fe80::/64 via ::, mon, 18:44:25
```
- mac table 확인
	- 11번 통해서 맥러닝 되어 있다
```
TiFRONT% show mac-table
  ageing-time 300
 -----------------------------------------------------------
    No  | VLAN |  PORT  |   MAC ADDRESS  | FWD/DIS | STATIC
 -------+------+--------+----------------+---------+--------
      1 |    1 |   ge11 | 0006.c452.1038 | FORWARD |
      2 |    1 |    CPU | 0006.c475.08bc | FORWARD | STATIC
      3 |   20 |    CPU | 0006.c475.08bc | FORWARD | STATIC
      4 | 4001 |    CPU | 0006.c475.08bc | FORWARD | STATIC
      5 |    1 |   ge11 | 0006.c475.0aca | FORWARD |
      6 |    1 |   ge11 | 8cb0.e950.e0c2 | FORWARD |
      7 |    1 |   ge11 | e4c7.6756.a85b | FORWARD |
 -----------------------------------------------------------
```
# mgmt 인터페이스 라우팅 테스트
## 테스트1
- SW1의 mgmt 포트에 PC 연결 후 통신 테스트
- mgmt에 ip할당한 경우 > mgmt 인터페이스에 ip 넣으면 그 인터페이스 ip로만 통신 가능
	- mgmt ip로는 통신이 되지만, 5002::1로는 통신 불가
	- ipv6 라우팅 추가해도 동일
```
C:\Users\USER>ping 2400:a0a0:1015:5002::15 -t

Ping 2400:a0a0:1015:5002::15 32바이트 데이터 사용:
2400:a0a0:1015:5002::15의 응답: 시간=3ms

2400:a0a0:1015:5002::15에 대한 Ping 통계:
    패킷: 보냄 = 1, 받음 = 1, 손실 = 0 (0% 손실),
왕복 시간(밀리초):
    최소 = 3ms, 최대 = 3ms, 평균 = 3ms

--------
C:\Users\USER>ping 2400:a0a0:1015:5002::1 -t

Ping 2400:a0a0:1015:5002::1 32바이트 데이터 사용:
대상 호스트에 연결할 수 없습니다.
대상 호스트에 연결할 수 없습니다.
```
- 설정
```
TiFRONT% show ipv6 route
IPv6 Routing Table
Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
       IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, I - IS-IS, B - BGP
Timers: Uptime

S     ::/0 [1/0] via 2400:a0a0:1015:5002::1, mgmt0, 00:58:57
C     ::1/128 via ::, loopback, 19:23:51
C     2400:a0a0:1015:5002::/64 via ::, mgmt0, 01:02:52
C     fe80::/64 via ::, mon, 19:23:35


TiFRONT% show mac-table
  ageing-time 300
 -----------------------------------------------------------
    No  | VLAN |  PORT  |   MAC ADDRESS  | FWD/DIS | STATIC
 -------+------+--------+----------------+---------+--------
      1 |    1 |   ge11 | 0006.c452.1038 | FORWARD |
      2 |    1 |    CPU | 0006.c475.08bc | FORWARD | STATIC
      3 |   20 |    CPU | 0006.c475.08bc | FORWARD | STATIC
      4 | 4001 |    CPU | 0006.c475.08bc | FORWARD | STATIC
      5 |    1 |   ge11 | 0006.c475.0aca | FORWARD |
 -----------------------------------------------------------
```
- vlan20에 ip할당한 경우
	- 연결된 인터페이스가 다르기 때문에 통신x
## 테스트2
- SW3 mgmt 포트에 PC 연결 후 통신 테스트 (G2728GP)
	- 갑자기 인터페이스와도 통신 안 됨
	- sw3 맥러닝 조회 시 pc 맥러닝 안 됨
```
TiFRONT% show mac-table
  ageing-time 300
 -----------------------------------------------------------
    No  | VLAN |  PORT  |   MAC ADDRESS  | FWD/DIS | STATIC
 -------+------+--------+----------------+---------+--------
      1 |    1 |    CPU | 0006.c452.1038 | FORWARD | STATIC
      2 |   20 |    CPU | 0006.c452.1038 | FORWARD | STATIC
      3 | 4001 |    CPU | 0006.c452.1038 | FORWARD | STATIC
      4 |    1 |   ge12 | 0006.c475.08bc | FORWARD |
      5 |    1 |   ge12 | 0006.c475.0aca | FORWARD |
 -----------------------------------------------------------
 
 C:\Users\USER>ping 2400:a0a0:1015:5001::8 -t

Ping 2400:a0a0:1015:5001::8 32바이트 데이터 사용:
요청 시간이 만료되었습니다.
요청 시간이 만료되었습니다.
요청 시간이 만료되었습니다.
요청 시간이 만료되었습니다.
```
- 설정
```
TiFRONT% sh ipv6 int b
ip6tnl0                    [administratively down/down]
    unassigned
loopback                   [up/up]
    ::1
mgmt0                      [up/up]
    2400:a0a0:1015:5001::8
    fe80::206:c4ff:fe52:1038
mgmtcs                     [up/up]
    fe80::206:c4ff:fe52:1038
mon                        [up/up]
    fe80::206:c4ff:fe52:1039
sit0                       [administratively down/down]
    unassigned
smon                       [up/up]
    fe80::206:c4ff:fe52:1039
vlan1                      [up/up]
    fe80::206:c4ff:fe52:1038
vlan20                     [up/up]
    fe80::206:c4ff:fe52:1038
TiFRONT%
TiFRONT%
TiFRONT% show ipv6 route
IPv6 Routing Table
Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
       IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, I - IS-IS, B - BGP
Timers: Uptime

C     ::1/128 via ::, loopback, 18:04:22
C     2400:a0a0:1015:5001::/64 via ::, mgmt0, 00:01:07
C     fe80::/64 via ::, mon, 18:04:04
```
- 재부팅 후
	- 동일 (통신 안 됨)
- 설정 초기화 후
	- 스위치에서 자기 자신의 ip로 통신 후 mgmt 인터페이스와 통신 되기 시작함
	- next hop (5001::1)로는 여전히 통신 안 됨
```
TiFRONT% ping ipv6 2400:a0a0:1015:5001::8
PING 2400:a0a0:1015:5001::8(2400:a0a0:1015:5001::8) 56 data bytes
64 bytes from 2400:a0a0:1015:5001::8: icmp_seq=1 ttl=64 time=0.209 ms
64 bytes from 2400:a0a0:1015:5001::8: icmp_seq=2 ttl=64 time=0.153 ms

--- 2400:a0a0:1015:5001::8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1015ms
rtt min/avg/max/mdev = 0.153/0.181/0.209/0.028 ms
TiFRONT% ping ipv6 2400:a0a0:1015:5001::11
PING 2400:a0a0:1015:5001::11(2400:a0a0:1015:5001::11) 56 data bytes
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=1 ttl=128 time=1.26 ms
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=2 ttl=128 time=0.950 ms
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=3 ttl=128 time=0.910 ms
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=4 ttl=128 time=0.983 ms
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=5 ttl=128 time=0.929 ms
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=6 ttl=128 time=0.872 ms
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=7 ttl=128 time=0.718 ms
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=8 ttl=128 time=0.674 ms

--- 2400:a0a0:1015:5001::11 ping statistics ---
8 packets transmitted, 8 received, 0% packet loss, time 7075ms
rtt min/avg/max/mdev = 0.674/0.912/1.262/0.169 ms
TiFRONT% ping ipv6 2400:a0a0:1015:5001::11
PING 2400:a0a0:1015:5001::11(2400:a0a0:1015:5001::11) 56 data bytes
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=1 ttl=128 time=0.750 ms
64 bytes from 2400:a0a0:1015:5001::11: icmp_seq=2 ttl=128 time=0.702 ms

--- 2400:a0a0:1015:5001::11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1011ms
rtt min/avg/max/mdev = 0.702/0.726/0.750/0.024 ms
TiFRONT%
TiFRONT%
TiFRONT% show mac-table
  ageing-time 300
 -----------------------------------------------------------
    No  | VLAN |  PORT  |   MAC ADDRESS  | FWD/DIS | STATIC
 -------+------+--------+----------------+---------+--------
      1 |    1 |    CPU | 0006.c452.1038 | FORWARD | STATIC
      2 |   20 |    CPU | 0006.c452.1038 | FORWARD | STATIC
      3 | 4001 |    CPU | 0006.c452.1038 | FORWARD | STATIC
      4 |    1 |   ge12 | 0006.c475.08bc | FORWARD |
      5 |    1 |   ge12 | 0006.c475.0aca | FORWARD |
 -----------------------------------------------------------
```
- 추가 확인
	- ndp 확인 시 mgmt0 인터페이스로 5001::8 (자신의 ip) 학습 안 되어 있음
	- mgmt0 인터페이스로 패킷이 들어오지 않는 것으로 보임
	- 이번에는 자기 자신의 ip로 쏴도 pc랑 통신 안 됨
```
TiFRONT% show ipv6 neighbors
 IPv6 Address                             MAC Address     Interface  Type
ff02::16                                 3333.0000.0016   mgmtcs     dynamic
ff02::16                                 3333.0000.0016   smon       dynamic
ff02::16                                 3333.0000.0016   mon        dynamic
ff02::16                                 3333.0000.0016   mgmt0      dynamic
ff02::16                                 3333.0000.0016   vlan20     dynamic
fe80::64a3:a156:846f:5174                8cb0.e950.e0c2   mgmt0      dynamic
ff02::16                                 3333.0000.0016   vlan1      dynamic
ff02::1:ff52:1039                        3333.ff52.1039   smon       dynamic
ff02::1:ff52:1039                        3333.ff52.1039   mon        dynamic
ff02::1:ff52:1038                        3333.ff52.1038   mgmtcs     dynamic
ff02::1:ff00:11                          3333.ff00.0011   mgmt0      dynamic
2400:a0a0:1015:5001::11                  8cb0.e950.e0c2   mgmt0      dynamic
ff02::1:ff52:1038                        3333.ff52.1038   vlan20     dynamic
::1                                      0000.0000.0000   loopback   dynamic
ff02::1:ff52:1038                        3333.ff52.1038   mgmt0      dynamic
ff02::1:ff00:1                           3333.ff00.0001   mgmt0      dynamic
ff02::2                                  3333.0000.0002   mgmt0      dynamic
2400:a0a0:1015:5001::8                   0000.0000.0000   loopback   dynamic
ff02::1:ff52:1038                        3333.ff52.1038   vlan1      dynamic
ff02::1:ff00:8                           3333.ff00.0008   mgmt0      dynamic


TiFRONT% show interface mgmt0
Interface mgmt0
  Hardware is Ethernet  Current HW addr: 0006.c452.1038
  Physical:0006.c452.1038  Logical:(not set)
  index 2 snmp-index 2 metric 1 mtu 1500 arp ageing timeout 3000
  <UP,BROADCAST,RUNNING,MULTICAST>
  VRF Binding: Not bound
  Bandwidth 1g
  inet 10.10.10.1/24 broadcast 10.10.10.255
  VRRP Master of :  VRRP is not configured on this interface.
  inet6 2400:a0a0:1015:5001::8/64
  inet6 fe80::206:c4ff:fe52:1038/64
    input packets 0, bytes 0, dropped 0, multicast packets 0
    input errors 0, length 0, overrun 0, CRC 0, frame 0, fifo 0, missed 0
    output packets 0, bytes 0, dropped 0
    output errors 0, aborted 0, carrier 0, fifo 0, heartbeat 0, window 0
    collisions 0
    
    
TiFRONT% show ipv6 route
IPv6 Routing Table
Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
       IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, I - IS-IS, B - BGP
Timers: Uptime

C     ::1/128 via ::, loopback, 05:15:36
C     2400:a0a0:1015:5001::/64 via ::, mgmt0, 00:10:15
C     fe80::/64 via ::, mon, 05:15:23
K     ff00::/8 via ::, mgmt0, 05:15:36
```