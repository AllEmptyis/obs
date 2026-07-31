## libvirt 네트워크 기본 구조
- 호스트 내부에서 가상 네트워크 환경을 생성하고, 가상머신을 가상 네트워크 환경에 연결
- libvirt  역할
	- 리눅스 커널이 제공하는 네트워크 기능을 libvirt 툴을 사용하여 구성할 수 있도록 함
	- 즉 아래의 기능은 리눅스 커널의 네트워크 기능
### 구성
- virbr0 (NAT 모드)
	- libvirt가 자동으로 생성하는 L2브릿지 (NAT모드에서 사용)
	- 해당 브릿지를 통해 vm이 내부 통신
- dnsmasq (NAT 모드인 경우에만)
	- 리눅스 기반 DHCP+DNS 프로세스 (virbr0에 제공)
		- libvirt가 vm에게 자동으로 ip할당 및 내부 dns 처리를 위해 백그라운드에서 해당 프로세스 실행
		- 리눅스 시스템 데몬과 다른 프로세스, libvirt가 죽으면 함께 사라지는 프로세스
- VM: 가상머신
### 네트워크 유형
- NAT (default)
	- 호스트만 외부 네트워크와 연결 / 가상머신은 내부 L2구간에 위치
	- 가상브릿지에 탭 인터페이스가 슬레이브 형태로 연결
	- 기본 네트워크 (virbr0)
		- 기본 IP: 192.168.122.0/24
		- 기본 MAC: `52:54:00:xx:xx:xx`
	- 내부 사설 IP를 가지고 있으며, 외부와 통신하기 위해서는 Source NAT 수행
		- iptables의 **POSTROUTING 체인**에서 수행
	- vm은 외부로 통신 가능 / 외부에서 vm으로 접근 불가
	- 기본적으로 virbr0 기반으로 하나의 서브넷을 가지며, dnsmasq가 관리하기 때문에 하나의 대역만 가짐  
- Bridge
	- L2스위치처럼 동작하는 가상 인터페이스(br0)에 여러 인터페이스 연결하는 방식 
	- 해당 br0은 물리 nic과 연결되어 있음
		- 외부 네트워크와 L2를 물리 nic을 통해 연결하는 방식
	- 브리지를 직접 생성해야 함
- isolated
	- vm간 통신만 가능 / 외부 통신 불가
- routed
	- 호스트가 게이트웨이 역할을 하면서 ...??
	- xml 정의 필요
- macvtap
	- VM을  호스트 NIC에 직접 연결
	- 호스트와는 통신 불가
		- 리눅스에서는 같은 NIC으로 들어온 패킷을 다시 자기자신에게 보내지 않기 때문
#### 구성 / 동작 방식
- NAT (default)
	- Host (virbr0: 192.168.122.1)
		- VM과 통신하는 가상 브리지, 게이트웨이 역할
	- NAT (iptables)
		- 호스트의 iptables에서 확인
		- 아웃바운드: 호스트 ip로 SNAT
		- 인바운드: 가상머신으로 전달
	- VM: 192.168.122.x
		- virbr0에 연결 된 내부 ip 할당
			- dnsmasq 프로세스가 dhcp역할
	- 외부 네트워크: 외부에서 직접 접근은 불가
		- 외부 접속 위해서는 포트포워딩 필요
		![[Pasted image 20250610103420.png|700|683x393]]
	- 동작 순서
		1. libvirt가 virbr0 브리지와 NAT 네트워크 자동 생성 (`기본 192.168.122.0/24`)
		2. dnsmasq가 virbr0에서 DHCP 서버로 동작
			- vm에게 ip  대역 할당
		3. vm 외부 통신
			- vm -> virbr0 -> 호스트로 전달
			- 호스트의 iptables 규칙에 따라 SNAT 수행 (가상머신의 ip를 호스트 ip로 nat)
			- 호스트에서 vm으로 응답패킷 도착
- bridge
```
   [ VM1 ]    [ VM2 ]
      │          │
      ▼          ▼
   [ tap0 ]   [ tap1 ] ← VM의 NIC (libvirt가 자동 생성)
       └──┬────┬──┘
          ▼    ▼
        [ br0 ]  ← 소프트웨어 브리지
          │
        [ eno1 ] ← 물리 NIC
          │
      외부 네트워크
```
## 네트워크 구성에 따른 Libvirt 동작
- virbr0, dnsmasq 프로세스는 nat 모드일 때만 자동 실행 된다
- [[Libvirt 동작 방식]]
### NAT 모드
- viribr0 브릿지 자동 생성
- vm에게 dhcp 역할을 해주는 `dnsmasq 프로세스` 자동 실행
	- `dnsmasq`는 libvirt가 NAT 네트워크를 생성할 때 실행하는 프로세스
	- 리눅스 서비스 데몬과 관계 없음 / libvirt가 죽으면 같이 사라진다
- [[IPTABLES & nftables|iptables]] 규칙 자동 생성
	- [[IPTABLES & nftables#chain|POSTROUTING]] (SNAT / MASQUERADE)
		- 외부로 나가는 패킷을 호스트 ip로 SNAT
	- FORWARD 필터링
		- vm과 외부 통신 할 수 있도록 허용 (vm->외부로 트래픽이 나갈 수 있도록)
	- INPUT 필터링
		- 가상머신이 dnsmasq로부터 dhcp 요청을 받을 수 있도록 허용
#### dnsmasq
- libvirt가 자동 실행하는 프로세스
- DHCP 서버 기능
	- virbr0 인터페이스에서 dhcp 범위를 열어서 vm에게 할당 (defualt.xml 파일 참조)
	- 할당 기록
		- `/var/lib/libvirt/dnsmasq/default.addn-hosts` 에 저장
		- 리스 사라지면 파일 삭제 됨
- 내부 DNS 서버 기능 (호스트명 - ip)
	- vm name으로 ping 가능
		- /etc/hosts에 host 명 등록 필요
	- 수동 등록
		- `/var/lib/libvirt/dnsmasq/default.hostsfile`
- DNS forwarding 기능 (외부 이름 해석)
	- 가상머신이 외부 dns 요청 시 (ex.`google.com`) 호스트 DNS 설정(`/etc/resolv.conf`)으로 라우팅 해주는 역할
		- xml에서 별도의 dns 지정도 가능
- 리스 관리 
- 고정 IP 설정 기능
	- MAC 주소 기반으로 특정 vm에 원하는 ip 할당
- PXE 부팅 지원 (자주 사용x)
#### xml 파일
- libvirt 설치, 실행 시 자동으로 default.xml 파일이 생성 된다. (`/etc/libvirt/qemu/networks/default.xml`)
	- 기본 NAT 설정 관련하여 수정하고 싶을 때 사용
#### iptables(nftables) 생성
- `LIBVIRT_PRT` 등 libvirt가 자동으로 iptables 체인을 생성
- 해당 체인은 NAT모드에서 동작하는 가상머신에 적용
	- firewalld(시스템 방화벽)에는 적용X
##### 체인 설명
- LIBVIRT_FWI (Forward Inbound)
	- 외부->VM
	- virbr0 인터페이스를 통해 외부에서 가상머신으로 들어오는 트래픽 처리 / 이미 연결 된 세션 외 나머지 리젝
- LIBVIRT_FWO (Forward Outbound)
	- VM->외부
	- 122.0 대역에서 virbr0을 통해 외부로 나가는 패킷 허용 / 나머지 리젝
- LIBVIRT_FWX (Forward Cross)
	- VM 간 통신 (모두 허용)
- LIBVIRT_INP (input체인)
- LIBVIRT_OUT (output체인)
### Bridge 모드
- [[KVM-Bridge 모드]]
## 동작 확인
### NAT 모드 iptables 확인
- 네트워크 관리를 위해서 libvirt가 iptabels 체인 자동 추가
	- 기본 흐름은 같고, 기본 체인에서 libvirt체인을 참조하는 구조로 삽입
```
[root@localhost ~]# iptables -L -n -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 604K  538M LIBVIRT_INP  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
2085K 3433M LIBVIRT_FWX  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
2085K 3433M LIBVIRT_FWI  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
 741K   38M LIBVIRT_FWO  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 902K 1358M LIBVIRT_OUT  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain LIBVIRT_FWI (1 references)
 pkts bytes target     prot opt in     out     source               destination         --->일반적인 NAT 환경에서는 크게 의미 없음 / vm ip로 직접 들어오는 패킷X
1344K 3395M ACCEPT     all  --  *      virbr0  0.0.0.0/0            192.168.122.0/24     ctstate RELATED,ESTABLISHED #생성된 연결에 속한 패킷만 허용
    0     0 REJECT     all  --  *      virbr0  0.0.0.0/0            0.0.0.0/0            reject-with icmp-port-unreachable

Chain LIBVIRT_FWO (1 references)
 pkts bytes target     prot opt in     out     source               destination         -----> vm의 ip가 192.168.122.0/24가 아닐 경우 외부로 나가는 트래픽 거부
 741K   38M ACCEPT     all  --  virbr0 *       192.168.122.0/24     0.0.0.0/0           
    0     0 REJECT     all  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-port-unreachable #reject 할 때 icmp 메시지 전송

Chain LIBVIRT_FWX (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  virbr0 virbr0  0.0.0.0/0            0.0.0.0/0           

Chain LIBVIRT_INP (1 references)
 pkts bytes target     prot opt in     out     source               destination         
 1623  140K ACCEPT     udp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
    0     0 ACCEPT     tcp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:53
  202 64676 ACCEPT     udp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            udp dpt:67
    0     0 ACCEPT     tcp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:67

Chain LIBVIRT_OUT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     udp  --  *      virbr0  0.0.0.0/0            0.0.0.0/0            udp dpt:53
    0     0 ACCEPT     tcp  --  *      virbr0  0.0.0.0/0            0.0.0.0/0            tcp dpt:53
  202 66256 ACCEPT     udp  --  *      virbr0  0.0.0.0/0            0.0.0.0/0            udp dpt:68
    0     0 ACCEPT     tcp  --  *      virbr0  0.0.0.0/0            0.0.0.0/0            tcp dpt:68
```
- NAT 테이블 조회
	- POSTROUTING target LIBVIRT_PRT: 나가는 패킷에 대해 전부 snat
	- Return 규칙
		- 멀티캐스트, 브로드캐스트로 가는 패킷은 return(상위 체인인 postrouting으로 다시 돌아감)
	- MASQUEADE 규칙
		- 마스커레이드: 출발지 ip를 호스트 ip로 nat 해주는 규칙
			- NAT 테이블에서 사용하는 taget 이름
		- 출발지: 192.168.122.0/24 목적지: !192.168.122.0/24(해당 ip가 아닌 것, 즉 외부 ip)
			- 즉 목적지가 외부일 때 nat
```
[root@localhost ~]# iptables -t nat -L -n -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 3416  260K LIBVIRT_PRT  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain LIBVIRT_PRT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    2   167 RETURN     all  --  *      *       192.168.122.0/24     224.0.0.0/24        #멀티캐스트 or 브로드캐스트 주소로 가는 패킷은 NAT X
    0     0 RETURN     all  --  *      *       192.168.122.0/24     255.255.255.255     
  334 20040 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535  #외부로 나갈 땐 masquerade
 1458  111K MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    3   252 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24    #나머지 프로토콜도 nat 수행
```
- iptables-save
```
[root@localhost ~]# iptables-save
# Generated by iptables-save v1.8.8 (nf_tables) on Fri Jun 13 13:47:28 2025

*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:LIBVIRT_PRT - [0:0]
-A POSTROUTING -j LIBVIRT_PRT
-A LIBVIRT_PRT -s 192.168.122.0/24 -d 224.0.0.0/24 -j RETURN
-A LIBVIRT_PRT -s 192.168.122.0/24 -d 255.255.255.255/32 -j RETURN
-A LIBVIRT_PRT -s 192.168.122.0/24 ! -d 192.168.122.0/24 -p tcp -j MASQUERADE --to-ports 1024-65535
-A LIBVIRT_PRT -s 192.168.122.0/24 ! -d 192.168.122.0/24 -p udp -j MASQUERADE --to-ports 1024-65535
-A LIBVIRT_PRT -s 192.168.122.0/24 ! -d 192.168.122.0/24 -j MASQUERADE
COMMIT
```
### dnsmasq 파일 확인
- `/var/lib/libvirt/dnsmasq`  파일 확인
	- 가상 네트워크 관리를 위한 설정 파일
	```
	[root@localhost dnsmasq]# ls
	default.addnhosts  default.conf  default.hostsfile  virbr0.macs  virbr0.status
	```
- default.conf
	- dnsmasq 설정 파일
		```
		[root@localhost dnsmasq]# cat default.conf
		##WARNING:  THIS IS AN AUTO-GENERATED FILE. CHANGES TO IT ARE LIKELY TO BE
		##OVERWRITTEN AND LOST.  Changes to this configuration should be made using:
		##    virsh net-edit default
		## or other application using the libvirt API.
		##
		## dnsmasq conf file created by libvirt
		strict-order
		pid-file=/run/libvirt/network/default.pid
		except-interface=lo
		bind-dynamic
		interface=virbr0
		dhcp-range=192.168.122.2,192.168.122.254,255.255.255.0
		dhcp-no-override
		dhcp-authoritative
		dhcp-lease-max=253
		dhcp-hostsfile=/var/lib/libvirt/dnsmasq/default.hostsfile
		addn-hosts=/var/lib/libvirt/dnsmasq/default.addnhosts
		```
- default.addhosts
	- dnsmasq에 추가할 수 있는 수동 호스트 정보 파일
- virbr0.status
	- 현재 활성화 된 VM들의 ip/mac/네트워크 상태 정보 (실시간)
	- MAC, IP, 인터페이스 상태 등이 json으로 기록
		```
		[root@localhost dnsmasq]# cat virbr0.status 
		[
		  {
		    "ip-address": "192.168.122.155",
		    "mac-address": "52:54:00:be:28:d3",
		    "client-id": "01:52:54:00:be:28:d3",
		    "expiry-time": 1750036642
		  },
		  {
		    "ip-address": "192.168.122.167",
		    "mac-address": "52:54:00:80:84:3c",
		    "client-id": "01:52:54:00:80:84:3c",
		    "expiry-time": 1750037861
		  }
		]
		```
#### host 등록
- hostname 변경
	- hostnamectl set-hostname `name`
- `/etc/hosts` 등록
	```
	192.168.122.155         test1
	
	[root@localhost ~]# ping test1
	PING test1 (192.168.122.155) 56(84) bytes of data.
	64 bytes from test1 (192.168.122.155): icmp_seq=1 ttl=64 time=0.477 ms
	64 bytes from test1 (192.168.122.155): icmp_seq=2 ttl=64 time=0.501 ms
	```
### xml 파일 조회
- /etc/libvirt/qemu/networks/default.xml
	- libvirt가 관리하는 네트워크 정의 파일
	- 네트워크 설정을 바꾸려고 한다면 해당 파일에서 설정해야 함
		- libvirt가 나머지 파일(dnsmasq)를 재구성하여 자동 반영
	```
	[root@localhost networks]# cat default.xml 
	<!--
	WARNING: THIS IS AN AUTO-GENERATED FILE. CHANGES TO IT ARE LIKELY TO BE
	OVERWRITTEN AND LOST. Changes to this xml configuration should be made using:
	  virsh net-edit default
	or other application using the libvirt API.
	-->
	
	<network>
	  <name>default</name>
	  <uuid>0462c429-f9f1-470d-b76c-85a632adc5a8</uuid>
	  <forward mode='nat'/>
	  <bridge name='virbr0' stp='on' delay='0'/>
	  <mac address='52:54:00:f8:ab:33'/>
	  <ip address='192.168.122.1' netmask='255.255.255.0'>
	    <dhcp>
	      <range start='192.168.122.2' end='192.168.122.254'/>
	    </dhcp>
	  </ip>
	</network>
	```
## DHCP 자동 할당 실패
- 원인
	- tap 인터페이스가 브릿지랑 연결이 안 된 상황
- TAP 인터페이스 상태 확인
	- ip link show vnet1
	- ip link set vnet1 master virbr0
	- bridge link show
		- 브릿지 연결 확인
```
[root@localhost ~]# ip link show vnet1
6: vnet1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether fe:54:00:be:28:d3 brd ff:ff:ff:ff:ff:ff -->state unknonw

[root@localhost ~]# ip link set vnet1 master virbr0
[root@localhost ~]# ip link set vnet8 master virbr0

[root@localhost ~]# bridge link show
6: vnet1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master virbr0 state forwarding priority 32 cost 100
13: vnet8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master virbr0 state learning priority 32 cost 100

[root@localhost ~]# ip link set dev vnet1 up
[root@localhost ~]# ip link set dev vnet8 up
```
- VM에서 인터페이스 up
	- nmcli con up enp1s0

