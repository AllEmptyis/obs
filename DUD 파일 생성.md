## 방법
- sas2008이 mpt3sas 모듈을 인식하지 못하는 문제로 인해 elrepo에서 호환되는 드라이버 사용 필요
- 그러나 해당 드라이버도 sas2008을 인식하지 못하는 것으로 확인
	- rocky9.6에서 문제 해결 (https://elrepo.org/bugs/view.php?id=1530&utm_source=chatgpt.com)
- 아래의 패키지에서 mpt3sas.ko 모듈 추출
	- `kmod-mpt3sas-48.100.00.00-1.el9_6.elrepo.x86_64.rpm`
- 설치 방법
	- 버전 세대만 같으면 (el9) 커널 버전 달라도 호환 됨 (kABI 모듈)
	- iodd에 로키9.2 dvd iso 넣고, 초기 설치 시 grub에서 사용할 dud 지정하여 설치 진행
## 생성 단계
- [ ] boot usb 준비
	- rocky9.2 boot iso 다운로드
- [ ] 

## 생성 과정
### DUD 생성
- 생성 할 vm(iso) 정보
	- rocky linux 9.2
	- 5.14.0-284.11.1.el9_2.x86_64
#### rpm 파일 다운로드
```
dnf install -y https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm

[root@localhost ~]# dnf --enablerepo=elrepo-testing download kmod-mpt3sas
Last metadata expiration check: 0:05:00 ago on Mon 02 Jun 2025 01:01:49 PM KST.
kmod-mpt3sas-48.100.00.00-1.el9_6.elrepo.x86_64.rpm
```
#### ko 파일 추출
- 모듈 버전
	- 5.14.0-570.12.1.el9_6.x86_64
```
[root@localhost ~]# rpm2cpio kmod-mpt3sas-48.100.00.00-1.el9_6.elrepo.x86_64.rpm | cpio -ivd
./etc/depmod.d/kmod-mpt3sas.conf
./lib/modules/5.14.0-570.12.1.el9_6.x86_64
./lib/modules/5.14.0-570.12.1.el9_6.x86_64/extra
./lib/modules/5.14.0-570.12.1.el9_6.x86_64/extra/mpt3sas
./lib/modules/5.14.0-570.12.1.el9_6.x86_64/extra/mpt3sas/mpt3sas.ko
./usr/share/doc/kmod-mpt3sas-48.100.00.00
./usr/share/doc/kmod-mpt3sas-48.100.00.00/GPL-v2.0.txt
./usr/share/doc/kmod-mpt3sas-48.100.00.00/greylist.txt
2324 blocks

[root@localhost ~]# ls
anaconda-ks.cfg  etc  kmod-mpt3sas-48.100.00.00-1.el9_6.elrepo.x86_64.rpm  lib  usr
```
#### DUD 생성 할 디렉터리
- boot, etc, lib 폴더는 지울 것 (잘못 만들어짐)
```
# 변수 설정
[root@localhost ~]# KVER="5.14.0-570.12.1.el9_6.x86_64"
[root@localhost ~]# ARCH="x86_64"
[root@localhost ~]# WORK="/dud"

# DUD 만들 디렉토리 생성 
[root@localhost ~]# mkdir -p "$WORK/ks-modules/$ARCH/$KVER"
[root@localhost ~]#
[root@localhost ~]# ls /dud
boot  etc  ks-modules  lib
```
#### DUD 생성 및 ISO 패키징
- 실제 서버에 설치 할 커널 버전이 다르기 때문에 폴더명을 커널 버전에 맞도록 변경
	- /dud/ks-modules/x86_64/5.14.0-284.11.1.el9_2.x86_64
```
## ko를 DUD 디렉에 복사
[root@localhost ~]# cp ./lib/modules/$KVER/extra/mpt3sas/mpt3sas.ko /dud/ks-modules/$ARCH/$KVER/mpt3sas.ko

## ko 파일 xz로 압축 (공식 DUD포맷)
[root@localhost ~]# xz -9 /dud/ks-modules/$ARCH/$KVER/mpt3sas.ko

## 폴더명 변경
[root@localhost 86_64]# mv 5.14.0-570.12.1.el9_6.x86_64 5.14.0-284.11.1.el9_2.x86_64

## 해당 ISO가 드라이버 디스크라고 표시
[root@localhost ~]# touch /dud/rhdd3

## ISO 패키징
[root@localhost ~]# xorriso -as mkisofs -o dd-mpt3sas-48.100-el9_6.iso -V OEMDRV -R -J /dud
(생략)
Writing to 'stdio:dd-mpt3sas-48.100-el9_6.iso' completed successfully.
```
#### iso 파일 생성 완료
- dd-mpt3sas-48.100-el9_6.iso
### 서버 설정
- grub 진입하여 아래와 같이 입력
	- install하는 화면에서 `tab 키`
	```
	linuxefi … quiet
	    inst.dd=OEMDRV
	```
## 실패1
### 원인
- usb에 들어 있는 iso 자동 마운트 안 됨
- 수동으로 mpt3sas.ko 모듈 인식
	- 설치하려 한 ko 모듈의 vermagic(어떤 커널에서 빌드 되었는지)이 다르기 때문에 적용 안 됨
		- invalid module format
		- 아래와 같은 vermagic을 가진 모듈을 사용해야 함.
			```
			vermagic: 5.14.0 SMP mod_unload modversions
			5.14.0-xxx.el9_* : 특정 스트림 전용
			```
	- `kmod-mpt3sas-48.100.00.00-1.el9_6.elrepo.x86_64.rpm` 커널 9.6에서만 호환
### 해결 방법