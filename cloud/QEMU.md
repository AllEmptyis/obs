## QEMU
- KVM 혹은 레드헷에 포함 된 여러 애플리케이션에서 사용하는 에뮬레이션 도구
- 디스크, 네트워크, 직렬/병렬 포트 등과 같은 하드웨어 가상화를 수행
	- **사용자공간에서 실행되는 프로세스**
	- VM 당 하나의 프로세스 형태로 실행된다
		- qemu-system-x86_64 -name ubuntu-vm -m 2048 -smp 2 ...
- 두 가지 모드 (pure emulation / KVM 가속 모드)
	- 단독으로 애뮬레이션 가능하지만, 하드웨어 지원 가상화 없이는 매우 느리다
- 가상머신에 대한 인터페이스 제공 역할
	- 즉 실제 유저 / vm의 모든 요청을 qemu가 1차적으로 처리
- 즉 KVM은 커널 공간에서 실행 / QEMU는 사용자 공간에서 실행

## 핵심 기술
1) CPU 에뮬레이션 및 가상화
	- 
2) 디바이스 에뮬레이션 (가상 하드웨어 제공)
3) vm의 I/O 요청 처리
4) vm 실행/제어
5) KVM과 협업
## 실행 방식
- **qemu-system-x86_64와 같은 실행 파일이 VM 하나당 하나의 QEMU 프로세스를 생성**
	- 이 프로세스가 게스트OS 애뮬레이션 or 가속 모드(KVM)을 실행
- 실행방법
	1) 직접 명령어 실행 (복잡)
	2) [[cloud/libvirt.md|libvirt]]를 사용하여 VM에게 요청
		- VM 시작,중지 등을 요청 (xml로 실행)
			xml 파일에 QEMU 실행을 위한 설정이 들어 있으며, libvirt가 xml을 파싱하여 QEMU를 실행하면 QEMU 프로세스가 백그라운드에서 실행 됨 

> [!NOTE] flow
> [사용자]
   ↓
[vm.xml 작성]
   ↓
[virsh create vm.xml 실행]
   ↓
[libvirt API → libvirtd 데몬]
   ↓
[QEMU 프로세스 생성]
   ↓
[KVM 모듈을 통해 가상화 가속]

- QEMU 프로세스는 게스트의 입력/명령을 처리, 따라서 해당 리소스에만 액세스 할 수 있는 제한 된 환경에서 실행 필요
- 따라서 아래와 같이 libvirt가 해당 역할을 수행해준다
![[Pasted image 20250430102557.png]]

## KVM/QEMU 아키텍처
- KVM/QEMU 가상화 환경
![[Pasted image 20250430103010.png|500]]
- KVM guest
	- file system and block devices: 디스크에 접근하려는 동작
	- drivers: virtio 드라이버 or 에뮬레이트 된 장치 드라이버
	- vcpu0 - vcpu n: 게스트의 가상 cpu / 실제 호스트 cpu와 매핑 됨
- QEMU
	- 게스트가 보낸 I/O요청을 받아서 처리 or 실제 호스트 자원에 전달

- iothread란?
	- QEMU에서 디스크, 네트워크 등의<u> I/O 작업을 하도록 하는 스레드</u>

- virtio 장치란?
	- qemu가 내부에 게스트 os가 실제 하드웨어처럼 인식하게 하기 위해 제공하는  장치
	- virtio-blk, virtio-net, virtio-scsi 등 
	- QEMU 내부에 C코드로 구현되어 있다
- virtio 기본 구조
```text
[게스트 OS]
  └─ VirtIO 드라이버 (virtio-blk, virtio-net 등)
        ↓
[QEMU 사용자 공간]
  └─ VirtIO 장치 모델 (virtio-blk-device.c 등)
        ↓
[호스트 자원]
  └─ 실제 파일, 네트워크 인터페이스 등
```

- 게스트 입장에서는 virtio 디스크, 네트워크 장치를 실제 장치처럼 인식하지만, 사실 내부적으로는 virtio 드라이버 임
- 게스트가 보내는 요청은 VirtQueue라는 공유 메모리 큐에 전달
- 해당 메모리 큐를 qemu가 감지하고, 실제 하드웨어 장치로 처리

```
게스트 OS
┌────────────┐
│ VirtIO 드라이버 │◀─ I/O 요청 (virtqueue)
└────────────┘
        ↓ 공유 메모리
QEMU
┌────────────┐
│ VirtIO 장치 모델 │─▶ 실제 호스트 자원 접근
└────────────┘
```





- libvirt > qemu > kvm 통신 구조
> [!NOTE] 통신 구조
> [사용자]
  ↓
[libvirt (virsh, virt-manager, etc)]
  ↓ (XML 파싱, 명령 조립)
[QEMU 실행됨 (user space)]
  ↓
QEMU → `/dev/kvm` 장치 파일에 ioctl 시스템 호출
  ↓
[커널 공간: KVM 모듈]
  ↓
→ 실제 메모리 할당, vCPU 스레드 생성 등 수행

- QEMU의 vCPU 생성 과정
	- /dev/kvm(KVM 커널 모듈)을 open()
	- ioctl (KVM_CREATE_VM) //kvm에 VM생성 요청
	- ioctl(KVM_CREATE_VCPU) //kvm이 스레드 생성 (vcpu생성)
	- ioctl(KVM_ALLOCATE_MEMORY) //게스트 메모리 매핑
	- 이후 vcpu가 KVM_RUN ioctl을 반복 호출하며 실행
- ioctl() 이란?
	- C코드 함수 (시스템 호출)
	```
	int kvm_fd = open("/dev/kvm", O_RDWR);
	ioctl(kvm_fd, KVM_CREATE_VM, ...);
	```


------
## 참고
[[https://jaeho.tistory.com/entry/KVM-vs-QEMU]]