## 목차
- [x] KVM 기본 설정 및 패키지 확인
- [ ] VM생성
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
## VM생성
### virt-install 사용
- [[libvirt 명령어#virt-install|virt install 명령어 참고]]
- 기본 인터페이스 확인 (libvirt 자동 생성)
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