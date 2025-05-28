## Libvirt 개념
- 가상화 플랫폼 관리 도구 / 레드헷의 오픈소스 프로젝트
- QEMU/KVM, Xen, VMware 등 다양한 하이퍼바이저를 작동시키기 위한 통합 API
- **즉 API를 통해서 VM을 생성/운영/삭제 가능**
- 하이퍼바이저를 사용해서 VM을 생성할 때는 각 하이퍼바이저에 맞는 API를 선택해야 함
	- 여러 하이퍼바이저에서 공용으로 사용 가능한 관리 API
	![[Pasted image 20250430104724.png]]

- 리눅스호스트 -libvirt - 하이퍼바이저 이러한 구조로 설치
- 프로그램이 아니기 때문에 libvirt를 사용하는 관리 app 필요
- 해당 관리 app을 사용하여 하이퍼바이저를 조종
- 완료 되면 하나의 api(livert)로 여러 하이퍼바이저 관리가 가능하다 (단, livbirt에는 각각의 하이퍼바이저에 맞는 드라이버가 설치되어 있어야 함)
![[Pasted image 20250430110533.png]] 

> [!NOTE]
> mgmt app이란? (사용자가 QEMU에게 명령을 내리는 도구)
> - ex) 오픈스택의 Nova 컴포넌트, virt-manger, virsh (libvert 제어도구)
> - KVM에서는 위의 virt-manager 같은 libvert 제어도구, KVM_QEMU를 제어 함 (물론 libvert없이도 사용 가능하지만 그러면 매우 불편)
> - libvert를 사용할 수 있게 해주는 app / libvert는 내부에서 KVM과 같은 하이퍼바이저를 제어

## 구성
- libvirt API
- libvirtd: 데몬
- virsh: CLI 명령어 도구
- virt-manager: GUI 기반 가상 머신 관리 툴

## 핵심 기능
1) 다양한 하이퍼바이저를 통합 된 방식으로 관리
2) 가상 머신 관리
	- XML 기반으로 제어
3) 가상 네트워크 구성 지원
4) 스토리지 풀 및 볼륨 관리
	- libvirt는 스토리지를 풀, 볼륨 단위로 관리
	- Pool: 물리적/논리적 그룹, LVM, NFS 마운트
	- volume: vm에 연결되는 디스크 단위, 이미지
5) 보안 및 접근 제어
	- libvirtd와 vm 통신 시 TLS 사용 가능
	- VM 간 접근 차단
6) 원격 관리 및 모니터링
	- 원격 VM도 제어 가능


## KVM, QEMU, libvirt 관계
- KVM: 리눅스 커널 모듈. Intel VT 등을 사용하여 하드웨어 가상화를 사용하여 실제적으로 vm의 cpu/메모리를 생성해주는 역할
- QEMU: 가상 머신 프로세스 (qemu-system-x86_64실행파일로 실행 됨)
- libvirt: VM 생성/삭제/스냅샷 등을 관리 (API로 kvm, qemu에게 명령을 내려줌)

> [!NOTE] 플로우
> [사용자]
   ↓
[Mgmt App: virsh, virt-manager, OpenStack Nova 등]
   ↓
[libvirt API → libvirtd 데몬]
   ↓
[QEMU 프로세스 실행]
   ↓
[KVM (커널 레벨 하이퍼바이저)]

> [!NOTE] 예시
> libvirt는 내부적으로 xml 파일 등을 기반으로 명령어를 생성하여 QEMU 호출
> xml 파일에는 vm 구성정보(cpu, 메모리, 디스크 네트워크 등) 그리고 이 xml 파일을 읽어서 qemu한테 전달
> ex) virsh create vm.xml
> 해당 명령을 실행하면 libvirt가 vm.xml을 파싱해서 내부적으로 QEMU 명령어 처리
> ex) qemu-system-x86_64 -enable-kvm -m 2048 -hda /var/lib/libvirt/images/myvm.qcow2 -net nic -net bridge=virbr0 ...
> virsh > libvirt API 호출 > xml 파싱 > QEMU 명령어 생성하여 QEMU 프로세스 실행 



## 원격 하이퍼바이저 제어
- 여러 대의 노드에서 다수의 vm을 관리할 때 libvirt가 주로 사용 됨
- 아래는 libvirt와 libvirt 데몬을 이용한 원격 하이퍼바이저 제어
	- **ssh, tcp 등 프로토콜을 통해 원격 머신에서 실행 중인 libvirtd 데몬에 접속하여 하이퍼바이저를 제어**
	- 즉 오픈스택과 같이 대규모 인프라의 중앙 집중 제어를 가능하게 함
![[Pasted image 20250430110904.png]]

- 원격노드에서는 libvirt 데몬이 실행되어야 함
- 즉 오픈스택 등 다양한 클라우드 플랫폼에서 libvert를 사용하여 vm 운영


---------------
## 참고
https://computing-jhson.tistory.com/89