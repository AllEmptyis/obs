# 일감
- https://redmine.piolink.com/issues/129448
- https://redmine.piolink.com/issues/138936
## 기능 개발 목적
- 클라우드 환경에서 vxlan을 통해 통신하기 위함
- 다른 vpc에 있는 TMS(보안솔루션) 장비에 ssl decrypt 기능을 사용하고자 개발
	- TMS로 KS가 패킷을 복제하여 보내주는 방식, 보통 TMS가 다른 VPC에 있을 가능성이 높기 때문에 전송 수단으로 vxlan을 사용

# 구성
- kvm 환경에서 pas-ks, vm을 각각 생성한다
	- 각각 VTEP이 되는 구조
	- VTEP은 vxlan을 캡슐화(encap,decap)하는 구간을 의미
- 원래 오픈스택 환경에서 하려고 했으나, 아래와 같은 문제 때문에 하지 못함
	1) ovn 구성이라 내부적으로 vtep이 들어가 있음, pas-ks가 그 역할을 가져오기 어려움
	2) pas-ks를 오픈스택의 외부망, vm을 kvm에 생성하여 테스트 해보려고 했으나
	   알 수 없는 이유로 오픈스택에서 vxlan 멀티캐스트가 안 되어 실패함 (확인 예정)
## 상세 동작 과정

# 환경 구성
- pas-ks: 192.168.211.206 (언더레이)
	- vxlan10: 10.10.10.206 (오버레이)
- vm(rocky): 192.168.211.228
	- vxlan10: 10.10.10.228
- vm2: 192.168.211.168
	- vxlan10: 10.10.10.168
- vxlan 설정: vni 10, udp 4789 사용 (각 vtep 간 동일하게만 맞추면 됨)
- 멀티캐스트 ip: 239.1.1.1

# 설정
## PAS-KS
- test 인터페이스를 생성하여 언더레이 ip를 여기에 할당한다
	- 원래 mgmt 인터페이스에 들어가게 되는데, mgmt로 vxlan 설정하려하면 오류가 생겨서 test 인터페이스로 옮김
```
switch# show interface

================================================================================
INTERFACE Configuration
 ------------------------------------------------------------------------------
  Name    Status MAC Address       IPv4 Address       DHCP    DHCP_IPv4 Description
  test    up     52:54:00:b7:c4:5c 192.168.211.206/24 disable
  vxlan10 up     0e:2c:6d:18:4c:ab 10.10.10.206/24    disable
  mgmt    up     52:54:00:b7:c4:5c                    disable
================================================================================
```
- test 인터페이스 확인
```
switch# show int test

================================================================================
  INTERFACE: test
 ------------------------------------------------------------------------------
    Name                 : test
    Status               : up
    MAC Address          : 52:54:00:b7:c4:5c
    MTU                  : 1500

    IP
       IPv4 Address       Broadcast       Overlapped
       192.168.211.206/24 192.168.211.255

    IPv6                 :

    RPF                  : default
    ARP-ignore           : 0
    ARP-announce         : 0
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
- vxlan 인터페이스 생성 (이름: vxlan10)
	- 모드는 다른 버전에서 유니캐스트도 사용 가능
```
switch(config)# tunnel
switch(config-tunnel)# show vxlan vxlan10

================================================================================
  VXLAN: vxlan10
 ------------------------------------------------------------------------------
    VXLAN Name           : vxlan10
    VNI                  : 10
    Port                 : 4789
    Mode                 : multicast
    Group-Address        : 239.1.1.1
    Local-Address        : 192.168.211.206
    Interface            : test
================================================================================
```
## 서버(가상머신)
- vxlan10 인터페이스 생성 후 vni, 멀티캐스트 그룹 등을 생성, 단 이 때 인터페이스는 ens3(언더레이 인터페이스)로 지정해야 됨
- mtu는 자동으로 1450으로 지정됨
```
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:e3:0f:75 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    inet 192.168.211.228/24 brd 192.168.211.255 scope global noprefixroute ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fee3:f75/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
5: vxlan10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 5a:bc:a2:4d:87:37 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.228/24 scope global vxlan10
       valid_lft forever preferred_lft forever
    inet6 fe80::58bc:a2ff:fe4d:8737/64 scope link
       valid_lft forever preferred_lft forever
```
- 방화벽에서 udp 포트 허용
	- 안하면 패킷이 vxaln10으로 못 감
```
[root@localhost ~]# systemctl start firewalld.service
[root@localhost ~]# sudo firewall-cmd --add-port=4789/udp --permanent

[root@localhost ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens3
  sources:
  services: cockpit dhcpv6-client ssh
  ports: 4789/udp
  protocols:
  forward: no
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
- 생성 명령어
```
ip link add vxlan10 type vxlan id 10 local 10.10.10.100 group 239.1.1.1 dev ens160 ttl 60 dstport 4789

ip addr add 50.50.50.10/24 dev vxlan10 

sudo ip link set vxlan10 up
```
## 호스트
- 멀티캐스트 스누핑 끄기
	- 테스트 편의상, 켜도 됨
- 멀티캐스트 쿼리어 켜기
	- 브리지가 igmp 쿼리를 보내도록
```
echo 0 > /sys/devices/virtual/net/br0/bridge/multicast_snooping
echo 1 > /sys/devices/virtual/net/br0/bridge/multicast_querier
```
# 결과
- pas-ks(206)에서 228로 ping test
	- ens3 mac: 52:54:00:e3:0f:75
	- pas-ks mac: 52:54:00:b7:c4:5c
		- vxlan10 mac: 0e:2c:6d:18:4c:ab
```
switch# ping 10.10.10.228
PING 10.10.10.228 (10.10.10.228) 56(84) bytes of data.
64 bytes from 10.10.10.228: icmp_req=1 ttl=64 time=0.379 ms
64 bytes from 10.10.10.228: icmp_req=2 ttl=64 time=0.319 ms
64 bytes from 10.10.10.228: icmp_req=3 ttl=64 time=0.349 ms
64 bytes from 10.10.10.228: icmp_req=4 ttl=64 time=0.236 ms
```
- 겉에는 언더레이 패킷이 감싸져 있고 내부에는 오버레이 패킷이 캡슐화 되어 있는 것을 확인 가능
```
[root@localhost ~]# tcpdump -nei ens3 udp
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens3, link-type EN10MB (Ethernet), capture size 262144 bytes


04:51:39.750712 52:54:00:b7:c4:5c > 52:54:00:e3:0f:75, ethertype IPv4 (0x0800), length 148: 192.168.211.206.24615 > 192.168.211.228.vxlan: VXLAN, flags [I] (0x08), vni 10
0e:2c:6d:18:4c:ab > 6a:7c:c7:57:50:d3, ethertype IPv4 (0x0800), length 98: 10.10.10.206 > 10.10.10.228: ICMP echo request, id 24738, seq 28, length 64
04:51:39.750787 52:54:00:e3:0f:75 > 52:54:00:b7:c4:5c, ethertype IPv4 (0x0800), length 148: 192.168.211.228.49452 > 192.168.211.206.vxlan: VXLAN, flags [I] (0x08), vni 10
6a:7c:c7:57:50:d3 > 0e:2c:6d:18:4c:ab, ethertype IPv4 (0x0800), length 98: 10.10.10.228 > 10.10.10.206: ICMP echo reply, id 24738, seq 28, length 64
04:51:40.474886 52:54:00:b7:c4:5c > 52:54:00:e3:0f:75, ethertype IPv4 (0x0800), length 148: 192.168.211.206.24615 > 192.168.211.228.vxlan: VXLAN, flags [I] (0x08), vni 10
```
- vxlan10
	- icmp 패킷을 디캡슐화 해서 받음
```
[root@localhost ~]# tcpdump -nei vxlan10
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vxlan10, link-type EN10MB (Ethernet), capture size 262144 bytes
04:55:21.073516 0e:2c:6d:18:4c:ab > 6a:7c:c7:57:50:d3, ethertype IPv4 (0x0800), length 98: 10.10.10.206 > 10.10.10.228: ICMP echo request, id 24985, seq 5, length 64
04:55:21.073544 6a:7c:c7:57:50:d3 > 0e:2c:6d:18:4c:ab, ethertype IPv4 (0x0800), length 98: 10.10.10.228 > 10.10.10.206: ICMP echo reply, id 24985, seq 5, length 64
04:55:22.095914 6a:7c:c7:57:50:d3 > 0e:2c:6d:18:4c:ab, ethertype ARP (0x0806), length 42: Request who-has 10.10.10.206 tell 10.10.10.228, length 28
04:55:22.096252 0e:2c:6d:18:4c:ab > 6a:7c:c7:57:50:d3, ethertype ARP (0x0806), length 42: Reply 10.10.10.206 is-at 0e:2c:6d:18:4c:ab, length 28
```
- 멀티캐스트 그룹 확인
	- 물리NIC을 통해 실제 패킷을 주고받기 때문에 ens3이 멀티캐스트 그룹에 가입되어 있음
```
[root@localhost ~]# ip maddr show
1:      lo
        inet  224.0.0.1
        inet6 ff02::1
        inet6 ff01::1
2:      ens3
        link  01:00:5e:00:00:01
        link  33:33:00:00:00:01
        link  33:33:ff:e3:0f:75
        link  01:00:5e:01:01:01
        inet  239.1.1.1
        inet  224.0.0.1
        inet6 ff02::1:ffe3:f75
        inet6 ff02::1
        inet6 ff01::1
6:      vxlan10
        link  33:33:00:00:00:01
        link  01:00:5e:00:00:01
        link  33:33:ff:57:50:d3
        inet  224.0.0.1
        inet6 ff02::1:ff57:50d3
        inet6 ff02::1
        inet6 ff01::1
```
- 다른 vm 추가 시에도 정상 동작 확인

----------------

# unicast 동작 확인
## 서버 설정
- vxlan 생성할 때 remote ip를 지정해서 생성
```
[root@localhost ~]# ip link add vxlan10 type vxlan id 10 local 192.168.212.61 remote 192.168.211.222 dev ens160 dstport 4789

[root@localhost ~]# ip addr add 10.10.10.61/24 dev vxlan10

[root@localhost ~]# ip link set vxlan10 up

[root@localhost ~]# bridge fdb show dev vxlan10
00:00:00:00:00:00 dst 192.168.211.222 via ens160 self permanent
```
## pas-ks 설정
```
switch(config)# show int vxlan10

================================================================================
  INTERFACE: vxlan10
 ------------------------------------------------------------------------------
    Name                 : vxlan10
    Status               : up
    MAC Address          : ea:28:80:d5:98:3d
    MTU                  : 1500

    IP
       IPv4 Address    Broadcast    Overlapped
       10.10.10.222/24 10.10.10.255

    IPv6                 :

    RPF                  : default
    ARP-ignore           : 0
    ARP-announce         : 0
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

switch(config)#
switch(config)# show int test

================================================================================
  INTERFACE: test
 ------------------------------------------------------------------------------
    Name                 : test
    Status               : up
    MAC Address          : 52:54:00:a1:9f:8a
    MTU                  : 1500

    IP
       IPv4 Address       Broadcast       Overlapped
       192.168.211.222/24 192.168.211.255

    IPv6                 :

    RPF                  : default
    ARP-ignore           : 0
    ARP-announce         : 0
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
- vxlan10 인터페이스 mtu는 1450임-> ks에서만 1500으로 보여주는 거 같다
	- shell에서 확인
```
10: vxlan10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether ea:28:80:d5:98:3d brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.222/24 brd 10.10.10.255 scope global vxlan10
       valid_lft forever preferred_lft forever
    inet6 fe80::e828:80ff:fed5:983d/64 scope link
       valid_lft forever preferred_lft forever
```
- vlxan 설정
```
switch(config)# tunnel
switch(config-tunnel)# vxlan vxlan10
switch(config-tunnel-vxlan[vxlan10])# current

================================================================================
  VXLAN: vxlan10
 ------------------------------------------------------------------------------
    VXLAN Name           : vxlan10
    VNI                  : 10
    Port                 : 4789
    Group-Address        :
    Remote-Address       : 192.168.211.61
    Local-Address        : 192.168.211.222
    Interface            : test
================================================================================
```
-----------
# vxlan 멀티캐스트 추가 테스트
- kvm 리눅스 호스트를 멀티캐스트 서버 역할을 하도록 해서 테스트
- kvm의 가상머신을 대역을 나눠서 멀티캐스팅 하도록
## 호스트 설정
- br1 추가
```
[root@localhost ~]# nmcli connection add type bridge ifname br1 con-name br1
Connection 'br1' (463e8b17-0961-4fd3-b354-38acdd84ddaf) successfully added.
[root@localhost ~]# nmcli connection modify br1 ipv4.addresses 10.10.20.1/24
nmcli connection modify br1 ipv4.method manual
[root@localhost ~]# nmcli connection up br1
Connection successfully activated (controller waiting for ports) (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/23)
```
- br1에 연결할 가상머신 인터페이스 변경
	- virsh edit rocky8.4
```
    <interface type='bridge'>
      <mac address='52:54:00:e3:0f:75'/>
      <source bridge='br1'/> <--브릿지 변경
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>

```
- 호스트를 멀티캐스팅 서버 역할 하도록 설정
- 패키지 설치
```
dnf install frr -y

systemctl enable frr 
systemctl start frr

# 멀티캐스트 포워딩 활성화
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo "net.ipv4.conf.all.mc_forwarding=1" >> /etc/sysctl.conf
sysctl -p
```
- vtysh 진입해서 igmp, pim 설정 (frr의 터미널 진입 명령어)
	- pim 데몬 비활성화 된 경우 vi /etc/frr/daemons 에서 pimd 설정 yes로 변경
```
[root@localhost ~]# vtysh

Hello, this is FRRouting (version 8.5.3).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

localhost.localdomain# con
localhost.localdomain(config)# int br0
localhost.localdomain(config-if)# ip pim
localhost.localdomain(config-if)# ip igmp
localhost.localdomain(config-if)# ex
localhost.localdomain(config)# int br1
localhost.localdomain(config-if)# ip pim
localhost.localdomain(config-if)# ip igmp
localhost.localdomain(config-if)# ex
localhost.localdomain(config)# ex
localhost.localdomain# ex

localhost.localdomain(config)# ip pim rp 192.168.211.100 224.0.0.0/4
//pim 설정
자기자신을 RP로 지정한다
```
## VM 설정
- vxlan 인터페이스 추가
```
[root@localhost ~]# ip link add vxlan10 type vxlan id 10 group 239.1.1.1 dev ens3 dstport 4789
[root@localhost ~]# ip link set vxlan10 up
[root@localhost ~]# ip addr add 10.10.10.2/24 dev vxlan10
```
- 구성
```
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:e3:0f:75 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    inet 10.10.20.2/24 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fee3:f75/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: vxlan10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether d2:c0:54:6d:a7:92 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.2/24 scope global vxlan10
       valid_lft forever preferred_lft forever
    inet6 fe80::d0c0:54ff:fe6d:a792/64 scope link
       valid_lft forever preferred_lft forever

```

