## IODD 사용법
### IODD 포맷
- exFAT 파일 시스템으로 포맷
- iodd_iso E: 파티션 > `_ISO` 폴더에 설치 할 iso 파일 업로드
- usb2.0
## 설치 순서
1. BIOS / 
2. raid 구성 모드 진입 (`F10`)
	- **iodd mode ODD mode로 변경** (dual mode로 설정 시 iodd기기의 디스크도 인식)
	- raid 구성
3. OS 부팅 / 설치
	- iodd에서 설치 할 iso 파일 마운트 후 설치 진행
4. 완료
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
	- 부팅 순서 변경
- `F12` : PXE boot
### OS 정보


## 해결 시도
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
	- [ ] 사용 모듈 확인
##### DUD 사용하여 설치 시도
- [x] 부트모드 UEFI / iso 라벨로 설정하여 설치 시도 (실패)
	- 부트모드 BIOS로 해야 함
	- 라벨로 설정하면 인식은 하지만 dud iso가 sas2008을 거부함. (에러 메시지 확인)
## 오류
### mpt3sas 모듈에서 SAS2008 컨트롤러 인식 불가
#### 문제 원인
- https://forums.rockylinux.org/t/mpt3sas-does-not-work-with-rockylinux-9/6935
- 설치 대상 서버(Dell)의 raid controller SAS2008는 mpt2sas 모듈과 호환 됨 
- Rocky에서는 mpt3sas 모듈만 지원
	- 구형 하드웨어를 인식하지 못하는 문제인 것으로 추정
#### 모듈 확인
- modinfo mpt2sas
	- 커널이 해당 모듈을 지원하는지 확인
- lsmod | grep mpt2sas
	- 현재 해당 모듈이 로드 되어 있는지
- modprobe mpt2sas
	- 수동으로 모듈 로드
#### 현재 커널 버전 확인
- uname -r
#### 모듈 다운로드
- dnf -y install
	```
	https://dl.rockylinux.org/vault/rocky/9.2/BaseOS/x86_64/kickstart/Packages/k/kernel-5.14.0-284.11.1.el9_2.x86_64.rpm
	https://dl.rockylinux.org/vault/rocky/9.2/BaseOS/x86_64/kickstart/Packages/k/kernel-modules-extra-5.14.0-284.11.1.el9_2.x86_64.rpm
	https://dl.rockylinux.org/vault/rocky/9.2/BaseOS/x86_64/kickstart/Packages/k/kernel-modules-5.14.0-284.11.1.el9_2.x86_64.rpm
	```
```
# ELRepo 설치 (한 번만)
dnf install -y https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm

# 현재 커널에 맞는 kmod-mpt2sas 설치
dnf --enablerepo=elrepo install -y kmod-mpt2sas
```
#### DUD 사용하여 설치 시도
##### 생성 방법
inst.dd=hd:LABEL=DD_MPT2SAS
or
init.dd=hd:/dev/sdb:/dd-mpt3sas-43.100.00.00-1.el9_5.iso
##### DUD 파일
- https://elrepo.org/linux/elrepo/el9/x86_64/RPMS/
- kmod-mpt3sas-43.100.00.00-6.el9_5.elrepo.x86_64.rpm
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