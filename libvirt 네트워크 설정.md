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
	- 기본 네트워크 (virbr0)
		- 기본 IP: 192.168.122.0/24
	- 내부 사설 IP를 가지고 있으며, 외부와 통신하기 위해서는 Source NAT 수행
		- iptables의 **POSTROUTING 체인**에서 수행
	- vm은 외부로 통신 가능 / 외부에서 vm으로 접근 불가
- Bridge
	- L2스위치처럼 동작하는 가상 인터페이스(br0)에 여러 인터페이스 연결하는 방식
	- 해당 br0은 물리 nic과 연결되어 있음
		- 따라서 vm과 호스트가 동일 네트워크에 존재
	- 브리지를 직접 생성해야 함
- isolated
	- vm간 통신만 가능 / 외부 통신 불가
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
		![[Pasted image 20250610103420.png|700|683x393]]
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
### NAT 모드
- viribr0 브릿지 자동 생성
- vm에게 dhcp 역할을 해주는 `dnsmasq 프로세스` 자동 실행
	- `dnsmasq`는 libvirt가 NAT 네트워크를 생성할 때 실행하는 프로세스
	- 리눅스 서비스 데몬과 관계 없음 / libvirt가 죽으면 같이 사라진다
- [[IPTABLES|iptables]] 규칙 자동 생성
	- [[IPTABLES#chain|POSTROUTING]] (SNAT / MASQUERADE)
		- 외부로 나가는 패킷을 호스트 ip로 SNAT
	- FORWARD 필터링
		- vm과 외부 통신 할 수 있도록 허용 (vm->외부로 트래픽이 나갈 수 있도록)
	- INPUT 필터링
		- 가상머신이 dnsmasq로부터 dhcp 요청을 받을 수 있도록 허용
#### dnsmasq 역할
- DHCP 서버 기능
	- virbr0 인터페이스에서 dhcp 범위를 열어서 vm에게 할당 (defualt.xml 파일 참조)
	- 할당 기록
		- `/var/lib/libvirt/dnsmasq/default.leases` 에 저장
- 내부 DNS 서버 기능 (호스트명 - ip)
	- vm name으로 ping 가능
	- DHCP와 연동되어 자동 등록 됨
	- 수동 등록
		- `/var/lib/libvirt/dnsmasq/default.hostsfile`
- DNS forwarding 기능 (외부 이름 해석)
	- 가상머신이 외부 dns 요청 시 (ex.`google.com`) 호스트 DNS 설정(`/etc/resolv.conf`)으로 라우팅 해주는 역할
		- xml에서 별도의 dns 지정도 가능
- 리스 관리
	- `default.leases`파일에서 확인 가능
	- vm이 어떤 mac, ip를 받았는지 확인
- 고정 IP 설정 기능
	- MAC 주소 기반으로 특정 vm에 원하는 ip 할당
- PXE 부팅 지원 (자주 사용x)
#### xml 파일 생성
- libvirt 설치, 실행 시 자동으로 default.xml 파일이 생성 된다. (`/etc/libvirt/qemu/networks/default.xml`)
	- 기본 NAT 설정 관련하여 수정하고 싶을 때 사용