## IODD 사용법
### IODD 포맷
- exFAT 파일 시스템으로 포맷
- iodd_iso E: 파티션 > `_ISO` 폴더에 설치 할 iso 파일 업로드
- usb2.0
## 설치 순서
1. 구성 모드 진입 (`F10`)
	- raid 구성 (필요 시)
	- OS deployment - iodd로 iso 마운트 후 OS 설치
2. OS 부팅 / 설치
## 서버/OS 버전 및 정보
### 서버 정보
- 모델명
	- Dell poweredge R210 II / 2011
- RAID Controller (Pcle adapter)
	- Dell PERC H200A
	- SAS2008 (컨트롤러 칩셋)
#### 단축키
- `F2` : sytem setup
	- 바이오스 setup, 부트모드 등 변경
- `F10` : system service
	- Lifecycle Controller 진입 
		- 델서버 내장 시스템 통합 관리 인터페이스
		- Raid 구성 / OS 배포 / 펌웨어 업데이트 등
- `F11` : boot manager
- `F12` : PXE boot
## 해결 시도
- [[Rocky linux9.3 DUD 생성]] 
	- 성공
- [[DUD 파일 생성]]
- [x] 서버에 기존 ubuntu 설치 (v22.04)
	- 그냥 설치하면 화면 안나옴 / graphical mode로 진입했음 (이유? 모름)
	- lsblk 확인 -> sda 가상디스크 잡힘
	- mpt3sas 모듈 사용 중
- [x] vm으로  ubuntu 설치
	- 설치 잘 됨 확인
- [x] centos7 설치
	- 성공 / mpt2sas 모듈 있음. (수동 로드x)
- [ ] centos6.8 설치
- [x] Rocky linux8.4 설치 - 실패
	- [x] 사용 모듈 확인 (mpt2sas 모듈 없음)
- DUD 사용하여 설치 시도
	- [x] 부트모드 UEFI / iso 라벨로 설정하여 설치 시도 (실패)
		- 부트모드 BIOS로 해야 함
		- 라벨로 설정하면 인식은 하지만 dud iso가 sas2008을 거부함. (에러 메시지 확인)
## 오류
### mpt3sas 모듈에서 SAS2008 컨트롤러 인식 불가
#### 문제 원인
- https://forums.rockylinux.org/t/mpt3sas-does-not-work-with-rockylinux-9/6935
- 설치 대상 서버(Dell)의 raid controller SAS2008는 mpt2sas 모듈과 호환
- RHEL8 이상부터는 mpt2sas 모듈 미지원
	- 구형 하드웨어를 인식하지 못하는 문제
	- RedHat계열 리눅스 OS 설치 불가
## 기타 확인
#### ISO에 모듈 포함 여부 확인
- **해당 방법은 initrd.img에  모듈이 포함되어 있는지만 확인**
	- .rpm에서 수동으로 로드 가능하면 설치 성공
- VM에서 ISO 마운트 하여 확인 방법
- VM 종료 상태에서 확인 할 iso image 연결 후 부팅 -> `/dev/sr0` 으로 자동 마운트
	- initrd.img: 리눅스 설치 시 부팅에 사용되는 초기 RAM 디스크 이미지
	- 내부에 커널 모듈 포함 되어 있음
```
#ISO 마운트
mkdir /mnt/iso
mount /dev/sr0 /mnt/iso
cd /mnt/iso/isolinux

#initrd.img 압축 풀기 위한 디렉 생성
mkdir /tmp/initrd
cd /tmp/initrd
cp /mnt/iso/images/pxeboot/initrd.img .

#압축 해제 - 이미지 포맷 확인 필요 (ztd -> cpio 해제)
zstdcat initrd.img | cpio -idmv

#모듈 확인
find . -name 'mpt2sas.ko'
```
##### 결과
- rockylinux 8.4
	- mpt2sas, mpt3sas 모듈 없음
- CentOS 7
	- mpt2sas, mpt3sas 모듈 없음
- CentOS 6.8
	- mpt3sas 모듈 포함
		```
		[root@localhost initrd]# find . -name 'mpt3sas'
		./modules/2.6.32-642.el6.x86_64/kernel/drivers/scsi/mpt3sas
		```
## 대안
#### IT 모드 전환 방법
- PERC200 
## 명령어
#### 모듈 확인
- modinfo mpt2sas
	- 커널이 해당 모듈을 지원하는지 확인
- lsmod | grep mpt2sas
	- 현재 해당 모듈이 로드 되어 있는지
- modprobe mpt2sas
	- 수동으로 모듈 로드
#### 현재 커널 버전 확인
- uname -r
#### 특정 모듈 위치 찾기
- find /lib/modules/$(uname -r) -name 'mpt3sas.ko'

- modinfo mpt3sas | grep filename
- 시스템 하드웨어 pci id 확인
	- lspci -nn
- cat /etc/redhat-release
- nmcli device reapply <인터페이스명>

## 참고
- iodd 원리
	- iodd는 일반 디스크이지만 서버가 실제 광학 cd/dvd와 같이 인식하도록 함
	- 따라서 그냥 디스크에 iso가 들어있는 거지만 iso 내부의 구조를 그대로 사용할 수 있음
- usb 부팅
	- usb는 일반 디스크이기 때문에 iso파일만 들어 있으면 인식하지 못함, 따라서 rufus같은 것을 사용하여 부팅 디스크로 만들어주어야 한다
	- iso 파일을 해체해서 usb에 부팅이 가능한 구조로 설치하는 것
