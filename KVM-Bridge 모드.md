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
```
- 브릿지 인터페이스 (br0)
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
eno1-slave  6eccf5c7-acbf-4901-9fe9-49a0d1150d15  ethernet  eno1
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
    link/ether d4:ae:52:ca:a2:98 brd ff:ff:ff:ff:ff:ff
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
