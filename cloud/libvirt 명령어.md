## libvirt 기반 CLI 툴
- virt-install (주로 설치 시 사용)
	- ISO 등으로 VM 생성
	- xml 사용하지 않고도 간편 생성 가능 (명령어 사용 시 내부적으로 xml 파일 생성)
- **virsh**
	- VM 관리 (시작/종료/삭제/스토리지 등 제어)
	- xml 기반 관리 시 사용
- virt-clone
	- VM 복제
	- `--original <기존 vm이름> --name <새 vm 이름> --file <새 디스크 경로>`
- virt-convert
	- vm 포맷 변환
- virt-manager
	- KVM GUI 관리 접속
- virt-viewer
	- vm 원격 접속
### virt-install
- --name `vm 이름`
- --memory `2048`
- --vcpu `num`
- --cdrom `로컬 iso 경로`
- --location `설치 트리 url`
	- 설치 트리란: 네트워크 설치용으로 구성 된 리눅스 배포 파일 구조 (images, repodata, packages 등 있음)
	- ex) `https://dl.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/`
	  `https://dl.rockylinux.org/pub/rocky/9/AppStream/x86_64/os/`
	- 보통 full dvd 로 설치 됨
- --network `network=default` 
	- NAT, 브릿지 등 네트워크 구성 설정
- --os-variant `generic`
	- OS 종류에 따라 최적화 된 방식으로 설치
	- 지원 OS 확인
		- `osinfo-query os`
- --graphics
	- vnc, none 등 그래픽 설정
- --no autoconsole
	- 자동으로 virt-viewer 띄우지 않고 설치만 진행
- --disk `디스크 경로,크기 지정`
	- path=`/var/lib/rocky9.qcow2`,size=`20`
	- 경로 지정하여 생성 시 자동으로 설정한 경로에 qcow2 생성 됨
- --extra-args""
### virsh
- 기본 구조
	- virsh `명령어` `VM name or id` `option`
- virsh list `--all`
	- VM 목록 / 상태 조회
- virsh `명령어` `name`
	- start: 시작
	- shutdown: 정상 종료
	- destroy: 강제 종료
	- reboot: 재부팅
	- undefine: vm 정의 삭제
- virsh define `vm.xml`
	- xml로 vm 정의 (등록)
- virsh create `vm.xml`
	- xml로 vm 생성+시작
- virsh dumpxml `name` > `vm.xml`
- virsh dumpxml `name`
	- 해당 VM의 xml 파일 조회
- virsh edit `name`
	- VM의 xml 파일 수정 (완료 후 자동으로 libvirt가 반영)
- virsh console testvm
	- 콘솔 접속
		- virt-install로 vm 생성 시 아래와 같이 옵션 넣어서 설정해주어야 함
		  만일 그렇게 안하면 추후 vm의 xml 파일 수정 + vm 커널에서 콘솔 관련 옵션 활성화 필요
	- --extra-args="console=ttyS0"
		- 주의 / cdrom 으로 설치 시 extra args 같이 사용 불가
```
나중에 vm에서 콘솔 활성화 방법

1)grub진입해서 해당 옵션 추가 (콘솔로 부팅 메시지 출력)
console=ttyS0,115200n8
2)systemd 서비스 활성화
systemctl enable --now serial-getty@ttyS0.service
-> 로그인 프롬프트가 ttys0 위에서 실행

참고 링크:
https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_virtualization/proc_opening-a-virtual-machine-serial-console_assembly_connecting-to-virtual-machines
```
- virsh domblklist `name`
	- vm의 디스크 확인
- virsh domiflist `name`
	- vm의 nic 확인
- virsh dominfo `name`
	- vm 현재 상태 정보 확인
- virsh vncdisplay `name`
- virsh net-dumpxml default
	- libvirt NAT 네트워크 설정 xml 확인
- virsh net-list --all
	-  NAT 네트워크 확인
- virsh net-dhcp-leases default
	- NAT 모드에서 dhcp로 할당 받은 ip 확인
-  virsh net-`start / destroy` default
	- NAT 네트워크 시작 / 강제 종료
- virsh dumpxml `myvm > /경로/myvm.xml`
	- 가상 머신 옮길 때 qcow2, xml 파일 백업 받은 후 virsh define 으로 xml 정의해주면
	  새로 처음부터 설치 할 필요x