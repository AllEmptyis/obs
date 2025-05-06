# KVM 개요
## KVM 개념/정의
- Kernel-based Virtual Machine의 약자로, 리눅스 커널에 내장된 하이퍼바이저 역할을 하는 커널 모듈
	- 커널 모듈일 뿐, 단독으로는 동작하지 못한다 (QEMU 등으로부터 명령을 받아야 함)
	- 커널 모듈 형태로 동작 /  해당 모듈은 cpu/메모리 가상화 담당
		- 가상 CPU 제공 및 vm마다 독립된 가상 메모리 공간 생성
		- CPU의 하드웨어 가상화 기능을 활성화/제어
## 구성
- KVM 커널 모듈 (CPU/메모리 가상화)
	- kvm.ko: kvm의 공통 기능을 담은 기본 커널 모듈
	- kvm-intel.ko: Intel CPU의 VT-x 기능 사용 시
- [[cloud/QEMU.md|QEMU (Quick EMUlator)]]
	- VM을 소프트웨어적으로 에뮬레이션하는 오픈소스 도구
	- 가상머신에 필요한 가상 디바이스(virtio, 디스크, usb  등) 에뮬레이션
	 -> 가상 디바이스가 있으면 (가상디스크 장치 등) vm에 내장 된 가상 드라이버가 해당 가상 디바이스를 읽고 쓸 수 있는 것 
	- 소프트웨어적으로 에뮬레이션 하는 경우, 속도가 매우 느리고 비효율적이기 때문에 주로 하드웨어 가상화를 지원하는 kvm과 함께 사용
	- 주요 역할
			- 장치 에뮬레이션(디스크, NIC 등)
			- vm 이미지 관리
			- vm의 GUI/CLI 화면 제공
- [[cloud/libvirt.md| Libvirt (관리도구)]]
## 사용
- openstack, GCP, IBM 클라우드 등

<kvm 구조>
![[Pasted image 20250429180735.png|300]]
