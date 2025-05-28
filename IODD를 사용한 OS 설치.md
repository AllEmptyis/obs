## IODD 사용법
### IODD 포맷
- exFAT 파일 시스템으로 포맷
- iodd_iso E: 파티션 > `_ISO` 폴더에 설치 할 iso 파일 업로드
- usb2.0 / dual mode 로 변경
## 설치 순서
1. 부트 모드 진입
	- iodd > 서버 disk
2. raid 구성 모드 진입 (`F10`)
	- **iodd mode ODD mode로 변경** / dual mode로 설정 시 iodd기기의 디스크도 인식  
	- raid1으로 구성
3. OS 부팅 / 설치
	- iodd 기기 넣고 재부팅 (이제 iodd에 있는 rocky linux가 인식 되어 로키로 부팅 됨)
	- 인식 된 설치 디스크가 올바른지 확인 후 설치 진행
4. 완료
## 해결 시도
- [ ] 서버에 기존 ubuntu 설치 (v22.04)
	- 그냥 설치하면 화면 안나옴 / graphical mode로 진입했음 (이유? 모름)
	- lsblk 확인 -> sda 가상디스크 잡힘
	- mpt3sas 모듈 사용 중
- [x] vm으로  ubuntu 설치
	- 설치 잘 됨 확인
- [ ] centos7 설치
- [ ] 부트모드 bios -> uefi 변경해서 부팅 시도
	- vm에서도 legacy로 부팅했기 때문에 가능성은 적음
## 오류
### mpt2sas 모듈 커널 미지원
- rockylinux 에서 해당 커널 모듈을 지원하지 않아 가상디스크가 보이지 않는 문제
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