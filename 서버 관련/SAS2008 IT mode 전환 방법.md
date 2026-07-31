## 작업 배경
- Dell 서버에 rocky linux 설치 중 가상디스크를 인식하지 못하는 문제 발생
	- RHEL 8 이상부터는 mpt2sas 모듈 지원x
- SAS2008은 Dell 서버에서 사용하는 **스토리지 컨트롤러 칩셋**
	- 디스크를 관리 / 연결해주는 역할 (레이드 등)
	- 리눅스에서 사용 시, 커널 드라이버 mpt2sas/mpt3sas를 통해 인식 됨
		- 따라서, mpt2sas 모듈
### 사용 모드
#### IT mode (initiator Target)
- 용도: HBA (host bus adapter)
	- 서버와 디스크(hdd)를 연결해주는 역할, 데이터를 직접 전달
	- RAID 기능 없고, OS가 디스크를 각각 따로 인식
#### IR mode (Intergrated RAID)
- 용도: 디스크를 raid로 구성하여 가상디스크로 제공
	- 레이드 구성 가능, OS에서는 하나의 가상디스크로 인식
## IT 모드 플래싱 방법
- SAS2008칩셋에 IR/IT 기능을 하는 펌웨어가 탑재되어 있음
	- 해당 펌웨어를 IR -> IT 모드 펌웨어로 플래싱(교체)
	- 바이너리를 덮어 씀
- 참고
	- https://www.2cpu.co.kr/lec/3116
	- https://bongtae.net/lsi-9211-8i-it-mode%EB%A1%9C-firmware-%EC%97%85%EB%8D%B0%EC%9D%B4%ED%8A%B8%ED%95%98%EA%B8%B0/
### 준비 사항
- DOS 부팅 가능한 usb
	- DOS 부팅이란? : Disk Operating System(디스크 운영 체제)가 컴퓨터를 시작하는 과정
		- FAT32 형식으로 포맷 되어야만 DOS가 인식 가능