## 목차
- [x] KVM 기본 설정 및 패키지 확인
- [ ] VM생성
	- [x] vm SSH 원격 접속
	- minimal 방식으로 설치
- xml 사용
- qcow2
- 네트워크
	- NAT 기본 ip 변경
## 기본 설정
- 모듈 확인
- 바이오스에서 VT-x 활성화 되어 있으면 자동으로 kvm 관련 모듈 로드
	- 모듈이 로드 되면 자동으로 `/dev/kvm` 장치 파일 생성
	```
	[root@localhost /]# egrep -c '(vmx|svm)' /proc/cpuinfo
	16 //가상화 지원
	
	[root@localhost /]# lsmod | grep kvm
	kvm_intel             479232  0
	kvm                  1327104  1 kvm_intel
	irqbypass              16384  1 kvm
	
	[root@localhost /]# ls -l /dev/kvm
	crw-rw-rw-. 1 root kvm 10, 232  6월  5 19:02 /dev/kvm
	//kvm.ko가 로드 되면 자동 생성되는 디바이스 파일

	[root@localhost lib]# find /lib/modules/$(uname-r) -name 'kvm*'
	bash: uname-r: command not found...
	/lib/modules/5.14.0-362.8.1.el9_3.x86_64/kernel/arch/x86/kvm
	/lib/modules/5.14.0-362.8.1.el9_3.x86_64/kernel/arch/x86/kvm/kvm-amd.ko.xz
	/lib/modules/5.14.0-362.8.1.el9_3.x86_64/kernel/arch/x86/kvm/kvm-intel.ko.xz
	/lib/modules/5.14.0-362.8.1.el9_3.x86_64/kernel/arch/x86/kvm/kvm.ko.xz
	//모듈 위치
	```
- 필수 패키지 설치
	- dnf install -y qemu-kvm libvirt virt-install virt-manager virt-viewer libvirt-daemon-kvm libvirt-daemon-config-network bridge-utils\
	- virt-install: 명령어 기반 가상 머신 생성 도구
	  virt-manager: gui 기반 가상 머신 관리자
- 데몬 실행
	- systemctl enable --now libvirtd
	- systemctl status libvirtd
	- qemu는 데몬이 아닌 프로세스 형태이기 때문에 할 필요 없음
## VM생성 / 확인
### virt-install 사용
- [[libvirt 명령어#virt-install|virt install 명령어 참고]]
- 기본 인터페이스 확인 (libvirt 자동 생성)
	- NAT 설정 시 호스트 / vm이 해당 인터페이스를 게이트웨이로 사용
	```
	virbr0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
	        ether 52:54:00:f8:ab:33  txqueuelen 1000  (Ethernet)
	        RX packets 603056  bytes 31933921 (30.4 MiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 1141144  bytes 3038141365 (2.8 GiB)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	```
- 기본 NAT 모드 / qcow2 자동 생성 / location으로 로키 9.5 설치트리 지정 
```
[root@localhost lib]# virt-install --name test1 --memory 2048 --vcpu 2 --location https://dl.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/ --network network=default --os-variant rocky9 --disk /qcow2/test1_rocky9.5.qcow2,size=20
```
- 결과
```

```
### VM 정보 확인
- 지정 경로에 자동으로 디스크 생성
```
[root@localhost qcow2]# ls
test1_rocky9.5.qcow2
```
- vm의 디스크 확인
```
[root@localhost qcow2]# virsh domblklist 2
 대상   소스
-------------------------------------
 vda    /qcow2/test1_rocky9.5.qcow2
```
- vm의 nic 확인
```
[root@localhost qcow2]# virsh domiflist 2
 연결장치   유형      소스      모델     MAC
------------------------------------------------------------
 vnet1      network   default   virtio   52:54:00:be:28:d3
```
- vm 현재 상태 정보 확인
	```
	[root@localhost qcow2]# virsh dominfo 2
	Id:             2
	이름:         test1
	UUID:           e35ac683-a2eb-45f1-a16a-0f69d3c26b17
	OS 유형:      hvm
	상태:         실행 중
	CPU:            2
	CPU 시간:     294.7s
	최대 메모리: 2097152 KiB
	사용된 메모리: 2097152 KiB
	영구적:      예
	Autostart:      비활성화됨
	관리 저장:  아니요
	모안 모델:  selinux
	보안 DOI:     0
	보안 레이블: system_u:system_r:svirt_t:s0:c102,c327 (enforcing)
	```
- 기본 네트워크 확인 (NAT)
	```
	[root@localhost test]# ip a
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	    inet 127.0.0.1/8 scope host lo
	       valid_lft forever preferred_lft forever
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
	    link/ether 52:54:00:be:28:d3 brd ff:ff:ff:ff:ff:ff
	    inet 192.168.122.155/24 brd 192.168.122.255 scope global dynamic noprefixroute enp1s0
	       valid_lft 2441sec preferred_lft 2441sec
	    inet6 fe80::5054:ff:febe:28d3/64 scope link noprefixroute 
	       valid_lft forever preferred_lft forever

	[root@localhost test]# ip route
	default via 192.168.122.1 dev enp1s0 proto dhcp src 192.168.122.155 metric 100 
	192.168.122.0/24 dev enp1s0 proto kernel scope link src 192.168.122.155 metric 100 
	```
### SSH 접속 활성화 (NAT 모드)
- 방화벽 설정
	- ssh 서비스 허용
- vm에서 sshd active 상태인지 확인
	- systemctl status sshd
- 호스트에서 아래 명령어로 접속
	- ssh `사용자명`@`vm ip`
- 접속 확인
	```
	[root@localhost ~]# ssh test@192.168.122.155
	The authenticity of host '192.168.122.155 (192.168.122.155)' can't be established.
	ED25519 key fingerprint is SHA256:ziJ6fMH2HXWrHs07mXPLhnmIUL9mlXSJHVPzeMKgTqo.
	This key is not known by any other names
	Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
	Warning: Permanently added '192.168.122.155' (ED25519) to the list of known hosts.
	test@192.168.122.155's password: 
	```
