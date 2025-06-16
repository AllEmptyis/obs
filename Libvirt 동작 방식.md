## 기본 구성
- 가상디스크 (실제 OS 저장)
- xml 파일
	- xml 파일 기반으로 qemu 실행
- libvirt 기본 제공 네트워크 정의 xml
## Libvirt 동작 흐름
- libvirt는 xml 파일을 기반으로 가상머신을 생성
- xml 파일을 정의해주면 > libvirt가 등록 > 해당 파일 기반 + 기본 네트워크 파일 참조해서 vm 실행
1. 사용자 입력
	- 사용자가 직접 xml 파일 생성
		- virsh define `<xml>`
		- virsh-install 등 툴 사용하여 xml 파일 생성 -> 해당 xml 파일이 libvirt에 정의 됨 (xml 생성 + libvirt에 등록)
			- virt-install(CLI), virt-manamerI(GUI)
2. 사용자의 요청을 받으면 libvirt가 해당 xml 파일을 등록
	- `/etc/libvirt/qemu/`하위에 등록
3. vm 실행 요청 시
	- 등록 된 xml 파일 기반으로 qemu 실행
		- `qemu-system-x86_64` 명령어로 내부적 호출
	- 디스크, ISO 파일 연결
		- `<disk>`정의 참조
	- 네트워크 연결
		- `<interface>` 참조
	- 콘솔 /그래픽 설정
	- VM 이름, mac 주소 등 설정
4. 가상머신 상태 관리
5. libvirtd, dnsmasq 프로세스 등 실행
## 디스크 생성
- 직접 xml 파일 작성하여 등록 시 수동으로 디스크 생성 필요
- virt-install / virt-manager 사용 시 자동 생성
## 네트워크
- xml 파일에서는 어떤 네트워크 파일을 참조할지 지정
- libvirt가 기본적으로 vm 생성 시 참고할 네트워크 파일을 가지고 있다
	- default.xml
- [[Libvirt 네트워크 설정#네트워크 구성에 따른 Libvirt 동작]]
## 가상머신 복사
- virt-clone 사용
	- 내부적으로 virsh dumpxml, qemu-img convert, virsh define을 한 번에 처리하는 방식
	- 복제 대상 vm의 디스크를 full copy하는 방식과 동일
- 템플릿 기반 (CoW)
	- 각 vm의 디스크는 베이스 디스크를 백킹파일로 참조 / 각 vm마다 xml 정의 파일 존재
	- 생성이 빠르지만 vm이 서로 디스크 공유하는 방식이라 실제 운영에선 부적절
## 주요 경로
- xml 정의 파일
	- `/etc/libvirt/qemu/vm.xml`
	- 가상머신 설정 변경 시 해당 파일 수정
- 디스크 이미지
	- `/var/lib/libvirt/images/vm.qcow2` (기본)
- 네트워크 정의
	- `/etc/libvirt/qemu/networks/default.xml`
	- xml 생성 시 참조 할 다른 네트워크 파일 지정 가능