## Bridge 모드란?
- 리눅스 커널의 가상 L2 스위치를 생성하여 여러 네트워크 인터페이스를 해당 L2 스위치(br)에 연결하여 같은 대역에서 통신하도록 하는 방식
- 브릿지 모드에서만 mac learning 기능 활성화
- STP 기능 (옵션)
### Bridge 동작 방식
- 브짓지에 여러 nic이 연결 되어 있는 구조 / 브릿지에 연결 된 nic들을 슬레이브 포트 라고 함
	- 기존 호스트 nic은 ip를 갖지 않고 브릿지가 호스트의 ip를 가짐
- 가상머신은 브릿지를 통해 같은 대역에 할당 되고, 외부 dhcp를 통해 ip 할당
	- dhcp가 없는 환경이라면 가상머신에서 각각 ip 수동 설정 필요
- 가상머신은 tap 인터페이스를 통해 브릿지와 연결
	- vm 내부에서는 vmnet 형태로 보임
- 브릿지가 mac 주소를 학습해서 어느 포트로 패킷을 포워딩 할지 결정
- 슬레이브 포트: 브릿지에서 포트 역할을 하는 네트워크 인터페이스
### Bridge 구성
- 통신 흐름
```
VM → tap0 (vnet0) → br0 → ens33 → 외부
```
- 구성
```
[ VM (eth0) ]
     │
  [vnet0] ← tap interface, VM용 NIC
     │
   [br0] ← 리눅스 bridge (가상 스위치)
   ├── vnet0 (VM)
   └── ens33 (호스트 NIC)
     │
[ 외부 네트워크 ]

--------------
VM NIC (ens3)
   ↓
vnetX
   ↓
br0 (가상 스위치)
   ↓
eno1 (물리 uplink)
   ↓
외부 네트워크 (라우터, 인터넷)

------------
┌────────────┐      ┌────────────┐
│   VM1      │      │   Host     │
│  192.168.193.101 │  │ 192.168.193.100 │
└──────┬─────┘      └──────┬─────┘
       │                  │
     vnet1              br0
       │                  │
       └──────┬────────────┘
              │
           eno1  (물리 NIC) ─────► 외부 네트워크
```
- 브릿지 인터페이스 (br0)
	- 내부 인터페이스 간 프레임 전달(처리) 역할만 수행 / IP는 브릿지가 가지고 있지만 실제 통신 위해서는 업링크(호스트 nic) 거쳐야 함
- 슬레이브 인터페이스
	- tap 인터페이스 (vmnetx)
		- vm이 실행될 때 libvirt가 자동으로 생성
		- **호스트에 생성 됨**
			- tap 인터페이스는 가상머신이 호스트와 연결 될 때 사용하는 인터페이스, 즉 nat 모드에서도 자동 생성 됨
	- eno1 (원래 호스트의 nic)
- MAC 주소 테이블 (브릿지 모드에서만 작동)
	- mac 기반 포워딩, l2 브로드캐스트
- STP 설정 (옵션)
## 사용 용도
- 가상화 환경
## Bridge 모드 구성 방법
1. br0(브릿지) 인터페이스 생성
2. 기존 호스트 nic을 br0에 연결 (슬레이브)
3. bridge 방식으로 vm 생성
4. DHCP or 수동으로 ip 할당
	- 호스트와 같이 외부 dhcp 서버로부터 할당
## 명령어
- nmcli
	- 네트워크매니저 데몬 명령어 / RHEL 기본적으로 네트워크매니저 사용
	- 영구 설정, 표준 방식
- ip 명령어 방식
	- 일회성
- ifcfg 설정 파일 방식
	- CentOS / RHEL 방식
### nmcli
- [[nmcli 명령어]]
- 주의 사항
	- 기존 nic에 설정 된 라우팅 등 사라질 수 있음
1. 브릿지 인터페이스 생성
	- nmcli connection add type bridge ifname br0 con-name br0
2. 물리 nic을 브릿지에 연결 - 슬레이브로 설정
	- nmcli connection add type ethernet ifname ens33 con-name ens33-slave master br0
		- 인터페이스 ens33을 br0의 슬레이브로 지정
		- 연결명은 ens33-slave
			- 네트워크 매니저가 브릿지-nic 연결을 위해 생성한 가상 연결
3. 브릿지에 ip 주소 설정
	- nmcli connection modify br0 ipv4.method manual/auto
		- 매뉴얼로 설정 시 가장 마지막에 설정할 것
	- nmcli connection modify br0 ipv4.addresses 192.168.0.100/24
	- nmcli connection modify br0 ipv4.gateway 192.168.0.1
	- nmcli connection modify br0 ipv4.dns 8.8.8.8
	- nmcli connection modify br0 +ipv4.routes "10.0.0.0/24 192.168.193.254"
		- 10.0.0.0/24 로 향하는 트래픽을 193.254로
4. 연결 활성화 (브릿지, 슬레이브 연결)
	- nmcli connection up ens33-slave
	- nmcli connection up br0
## 실습
- bridge 생성
```
[root@localhost dnsmasq]# nmcli connection add type bridge ifname br0 con-name br0
'br0' (ca5053ed-3555-4796-9970-73f9e8922bf5) 연결이 성공적으로 추가되었습니다.

14: br0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether b2:72:77:a3:7b:27 brd ff:ff:ff:ff:ff:ff
```
- eno1을 slave로 설정
```
root@localhost dnsmasq]# nmcli connection add type ethernet ifname eno1 con-name eno1-slave master br0

[root@localhost dnsmasq]# nmcli conn show
NAME        UUID                                  TYPE      DEVICE 
br0         ca5053ed-3555-4796-9970-73f9e8922bf5  bridge    br0    
lo          232150f5-d752-450d-b748-d2c8a65af4f0  loopback  lo     
eno1        0b1cfdb4-035e-4e56-8fb5-5ab869b1c277  ethernet  eno1   
virbr0      0c33f793-b994-4119-b642-287f2cf898d8  bridge    virbr0 
vnet1       1f07e40f-530a-43c9-beff-a0aa08e42a04  tun       vnet1  
vnet8       dacc48ee-3214-47c6-ac95-b8fa62634109  tun       vnet8  
eno1        9f41de35-b61c-3ea5-8906-91669ca4e06e  ethernet  --     
eno1-slave  6eccf5c7-acbf-4901-9fe9-49a0d1150d15  ethernet  --     
eno2        6b06a16c-694e-492c-897c-bf36042d0a00  ethernet  --   

[root@localhost dnsmasq]# nmcli device show eno1
GENERAL.DEVICE:                         eno1
GENERAL.TYPE:                           ethernet
GENERAL.HWADDR:                         D4:AE:52:CA:A2:98
GENERAL.MTU:                            1500
GENERAL.STATE:                          100 (연결됨 (외부))
GENERAL.CONNECTION:                     eno1  --->conn up 하지 않아서 아직 활성화 되지 않음
```
- br0에 ip 주소 설정
```
root@localhost dnsmasq]# nmcli conn modify br0 ipv4.addresses 192.168.193.100/24
[root@localhost dnsmasq]# nmcli conn modify br0 ipv4.gateway 192.168.193.1
[root@localhost dnsmasq]# nmcli conn modify br0 ipv4.dns 8.8.8.8
[root@localhost dnsmasq]# nmcli conn modify br0 ipv4.method manual
+
nmcli conn modify br0 +ipv4.routes "0.0.0.0/0 192.168.193.1" 
```
- 연결 활성화
```
nmcli con up br0
nmcli con up eno1-slave
```
- 확인
```
[root@localhost ~]# nmcli conn show
NAME        UUID                                  TYPE      DEVICE
br0         ca5053ed-3555-4796-9970-73f9e8922bf5  bridge    br0
lo          232150f5-d752-450d-b748-d2c8a65af4f0  loopback  lo
virbr0      0c33f793-b994-4119-b642-287f2cf898d8  bridge    virbr0
vnet1       1f07e40f-530a-43c9-beff-a0aa08e42a04  tun       vnet1
vnet8       dacc48ee-3214-47c6-ac95-b8fa62634109  tun       vnet8
eno1-slave  6eccf5c7-acbf-4901-9fe9-49a0d1150d15  ethernet  eno1   ---->eno1 연결 끊김 및 eno1-slave up
eno1        9f41de35-b61c-3ea5-8906-91669ca4e06e  ethernet  --
eno2        6b06a16c-694e-492c-897c-bf36042d0a00  ethernet  --

[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether d4:ae:52:ca:a2:98 brd ff:ff:ff:ff:ff:ff  ----> ip 삭제
    altname enp2s0f0
3: eno2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether d4:ae:52:ca:a2:99 brd ff:ff:ff:ff:ff:ff
    altname enp2s0f1
4: virbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:f8:ab:33 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
6: vnet1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master virbr0 state UNKNOWN group default qlen 1000
    link/ether fe:54:00:be:28:d3 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:febe:28d3/64 scope link
       valid_lft forever preferred_lft forever
13: vnet8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master virbr0 state UNKNOWN group default qlen 1000
    link/ether fe:54:00:ca:df:9f brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:feca:df9f/64 scope link
       valid_lft forever preferred_lft forever
14: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether d4:ae:52:ca:a2:98 brd ff:ff:ff:ff:ff:ff
    inet 192.168.193.100/24 brd 192.168.193.255 scope global noprefixroute br0
       valid_lft forever preferred_lft forever
    inet6 fe80::a1fd:dcfe:1919:ab3c/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```
- 문제?
	- 서버 xvnc 원격 접속 끊김
### 브릿지 모드 vm 생성
- NAT 모드로 생성 된 vm의 xml 파일 수정
	- `/etc/libvirt/qemu/xml파일`
- virsh define /etc/libvirt/qemu/test2.xml
```
<interface type='network'>
  <source network='default'/>
  <model type='e1000'/>
</interface>

(수정)
<interface type='bridge'>
  <mac address='52:54:00:80:84:3c'/>
  <source bridge='br0'/>
  <model type='e1000'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>

[root@localhost qemu]# virsh define /etc/libvirt/qemu/test2.xml
test2에서 정의된 도메인 '/etc/libvirt/qemu/test2.xml'

[root@localhost qemu]# virsh start test2

[root@localhost qemu]# virsh domiflist test2
 연결장치   유형     소스   모델    MAC
-------------------------------------------------------
 vnet10     bridge   br0    e1000   52:54:00:80:84:3c
```
- VM에도 ip 설정 / 라우팅 추가 필요
	- 호스트와 별개이므로 따로 설정 필요
```
ip addr add 192.168.193.101/24 dev ens3
ip route add default via 192.168.193.1

[root@test2 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:80:84:3c brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    inet 192.168.193.101/24 scope global ens3
       valid_lft forever preferred_lft forever
```
- 문제
	- NAT모드 일 때 할당 받은 ip가 계속 존재
	- NIC, MAC주소가 같아서 리스가 재사용 되는 것으로 추정
		- 이전에 virt clone으로 복제했던 가상머신의 호스트명이 서로 같아서 생긴 문제로 확인
```
[root@localhost qemu]# virsh net-dhcp-leases default
 만료 시간             MAC 주소            통신규약   IP 주소              호스트이름   클라이언트 ID 또는 DUID
-----------------------------------------------------------------------------------------------------------------
 2025-06-17 11:20:52   52:54:00:be:28:d3   ipv4       192.168.122.155/24   -            01:52:54:00:be:28:d3
 2025-06-17 11:11:34   52:54:00:ca:df:9f   ipv4       192.168.122.133/24   test2        01:52:54:00:ca:df:9f

[root@test2 ~]# nmcli conn show
NAME  UUID                                  TYPE      DEVICE
lo    f63fd3b4-10c0-4a8c-a05d-10ce350c2c7d  loopback  lo
ens3  c8973d71-ec3c-4b2c-b1b5-1cdaab1ca5a3  ethernet  ens3
ens3  42b9e8b4-1853-30ed-a87a-fe7775c0069c  ethernet  --   ---> 해당 ens3로 nat모드에서 할당받은 ip가 계속 있는 것을 확인

[root@test2 ~]# nmcli connection delete 42b9e8b4-1853-30ed-a87a-fe7775c0069c
Connection 'ens3' (42b9e8b4-1853-30ed-a87a-fe7775c0069c) successfully deleted
```
- mac addr 변경 후 재시작
	- 아직 dhcp 임대에 ip가 남아있으나 활성화 되진 않음
```
[root@test2 ~]# nmcli conn show
NAME                UUID                                  TYPE      DEVICE
lo                  11ae8122-c5db-45c5-8550-26d3c1d238b7  loopback  lo
ens3                0e0fff17-29d6-41b6-b49b-8db690f0fbad  ethernet  ens3
Wired connection 1  6ef7e131-cda9-36b1-8cc9-d089d8fea55a  ethernet  --   --->mac 주소 바뀐 경우 네트워크매니저가 자동 생성
```
### 통신 확인
- br0은 L2 스위치 역할, 탭 인터페이스는 br0에 연결하는 포트(케이블)과 같은 역할
	- eno1은 탭 인터페이스X, 물리 nic
	- eno1은 물리 nic 역할만 하고, 브릿지가 ip를 가지고 패킷 처리 등 역할 수행
- VM -> 호스트 ip로 ping (같은 브리지 내부)
	- 내부 통신이기 때문에 브릿지가 처리
	- vm -> br0
```
[root@localhost qemu]# tcpdump -ni vnet11
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on vnet11, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:00:30.187189 ARP, Request who-has 0.0.0.0 (Broadcast) tell 0.0.0.0, length 46
11:00:30.668113 IP 192.168.193.2 > 224.0.0.18: VRRPv2, Advertisement, vrid 193, prio 101, authtype none, intvl 1s, length 20
11:00:30.855751 IP 192.168.193.101 > 192.168.193.100: ICMP echo request, id 3, seq 5, length 64
11:00:30.855774 IP 192.168.193.100 > 192.168.193.101: ICMP echo reply, id 3, seq 5, length 64

[root@localhost qemu]# tcpdump -ni br0 -v icmp
dropped privs to tcpdump
tcpdump: listening on br0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:03:27.943787 IP (tos 0x0, ttl 64, id 29363, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.193.101 > 192.168.193.100: ICMP echo request, id 4, seq 61, length 64
11:03:27.943818 IP (tos 0x0, ttl 64, id 6305, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.193.100 > 192.168.193.101: ICMP echo reply, id 4, seq 61, length 64
11:03:28.967751 IP (tos 0x0, ttl 64, id 29784, offset 0, flags [DF], proto ICMP (1), length 84)

[root@localhost qemu]# tcpdump -ni eno1 -v icmp
dropped privs to tcpdump
tcpdump: listening on eno1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```
- VM -> 외부 통신 (8.8.8.8 ping)
	- 외부통신이기 때문에 eno1 을 거치게 됨
	- vm -> br0 -> eno1 -> 외부 
```
[root@localhost qemu]# tcpdump -ni br0 -v icmp
dropped privs to tcpdump
tcpdump: listening on br0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:22:59.566884 IP (tos 0x0, ttl 64, id 20309, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.193.101 > 8.8.8.8: ICMP echo request, id 5, seq 6, length 64
11:22:59.598560 IP (tos 0x0, ttl 106, id 0, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.193.101: ICMP echo reply, id 5, seq 6, length 64
11:23:00.568046 IP (tos 0x0, ttl 64, id 20917, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.193.101 > 8.8.8.8: ICMP echo request, id 5, seq 7, length 64

[root@localhost qemu]# tcpdump -ni eno1 -v icmp
dropped privs to tcpdump
tcpdump: listening on eno1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:23:06.576768 IP (tos 0x0, ttl 64, id 23661, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.193.101 > 8.8.8.8: ICMP echo request, id 5, seq 13, length 64
11:23:06.608539 IP (tos 0x0, ttl 106, id 0, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.193.101: ICMP echo reply, id 5, seq 13, length 64
11:23:07.577952 IP (tos 0x0, ttl 64, id 24349, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.193.101 > 8.8.8.8: ICMP echo request, id 5, seq 14, length 64
11:23:07.609502 IP (tos 0x0, ttl 106, id 0, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.193.101: ICMP echo reply, id 5, seq 14, length 64

[root@localhost qemu]# tcpdump -ni vnet11 -v icmp
dropped privs to tcpdump
tcpdump: listening on vnet11, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:23:20.597153 IP (tos 0x0, ttl 64, id 30124, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.193.101 > 8.8.8.8: ICMP echo request, id 5, seq 27, length 64
11:23:20.628684 IP (tos 0x0, ttl 106, id 0, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.193.101: ICMP echo reply, id 5, seq 27, length 64
11:23:21.599031 IP (tos 0x0, ttl 64, id 30954, offset 0, flags [DF], proto ICMP (1), length 84)
```