## rocky linux 9.3에서 dud 만들기
- sas2008 pci id를 지원하는 mpt3sas 모듈을 iso 파일로 만들어서 부팅 할 때 해당 드라이버를 사용하도록 하는 방법
- 조건
	- 설치하려는 iso와 kmod mpt3sas가 빌드 된 커널 버전이 같아야 함
	- kmod rpm에 pci id`1000:0072` 지원하는지 미리 확인
- 사용 rpm
	- [kmod-mpt3sas-43.100.00.00-3.el9_3.elrepo.x86_64.rpm](https://elrepo.org/linux/elrepo/el9/x86_64/RPMS/kmod-mpt3sas-43.100.00.00-3.el9_3.elrepo.x86_64.rpm)
	- dnf download 
- 현재 커널 버전
	- 5.14.0-362.8.1
- rpm 다운로드  후 ko 모듈 위치
	- /lib/modules/`version`/extra/mpt3sas/mpt3sas.ko
- 해당 ko 모듈을 iso 파일 만들 곳으로 복사 후 iso 파일로 생성
	- [[Rocky linux9.3 DUD 생성#올바른 iso 구조|파일 구조 참조]]
### dud 디렉토리 생성
```
[root@localhost dud]# KVER="5.14.0-362.8.1.el9_3.x86_64"
[root@localhost dud]# ARCH="x86_64"
[root@localhost dud]# mkdir -p "/dud/ks-modules/$ARCH/$KVER"
```
### mpt3sas.ko 모듈 복사
```
[root@localhost dud]# touch /dud/rhdd3
[root@localhost dud]# cp /home/test/lib/modules/$KVER/extra/mpt3sas/mpt3sas.ko /dud/ks-modules/$ARCH/$KVER/mpt3sas.ko

#확인
[root@localhost dud]# ls ks-modules/$ARCH/$KVER
mpt3sas.ko
```
### iso 패키징
```
[root@localhost dud]# xorriso -as mkisofs -o dd-mpt3sas-43.100-el9_3.iso -V OEMDRV -R -J /dud
xorriso 1.5.4 : RockRidge filesystem manipulator, libburnia project.

Drive current: -outdev 'stdio:dd-mpt3sas-43.100-el9_3.iso'
Media current: stdio file, overwriteable
Media status : is blank
Media summary: 0 sessions, 0 data blocks, 0 data, 10.7g free
Added to ISO image: directory '/'='/dud'
xorriso : UPDATE :       5 files added in 1 seconds
xorriso : UPDATE :       5 files added in 1 seconds
ISO image produced: 751 sectors
Written to medium : 751 sectors at LBA 0
Writing to 'stdio:dd-mpt3sas-43.100-el9_3.iso' completed successfully.

#확인
[root@localhost dud]# ls
dd-mpt3sas-43.100-el9_3.iso  ks-modules  rhdd3
```

## 오류
- iso 디렉토리 구조 잘못 생성하여 아나콘다가 인식 못함
#### 올바른 iso 구조
- /dud/rhdd3
- /dud/rpms/x86_64/`kmod rpm 파일`
- **createrepo_c dud/rpms/x86_64** (명령어 입력)
	- repodata 생성 필요
#### 서버
- `tab`키로 grub 모드 진입
- inst.dd 입력
- iso 파일을 옮긴 usb를 마운트 -> 생성한 kmod mpt3sas iso를 선택
## 확인
- 설치 후 파일 위치
	```
	[root@localhost /]# modinfo mpt3sas
	filename:       /lib/modules/5.14.0-362.8.1.el9_3.x86_64/extra/mpt3sas/mpt3sas.ko

	//extra에 있으면 외부 모듈이라는 뜻
	```
- dracut -f
- 이후 커널 업데이트 / OS 버전 업데이트 시에는 해당 버전에 맞는 kmod 를 사용해야 한다
	- **dnf update 시 커널도 같이 업데이트 되기 때문에 주의**
#### 조치
- dnf install -y dnf-plugins-core
  dnf install -y "dnf-command(versionlock)"
- dnf versionlock add kernel\*
	```
	[root@localhost network-scripts]# dnf versionlock list
	마지막 메타자료 만료확인(1:38:51 이전): 2025년 06월 09일 (월) 오전 09시 54분 32초.
	kernel-modules-core-0:5.14.0-362.8.1.el9_3.*
	kernel-0:5.14.0-362.8.1.el9_3.*
	kernel-tools-0:5.14.0-362.8.1.el9_3.*
	kernel-modules-0:5.14.0-362.8.1.el9_3.*
	kernel-tools-libs-0:5.14.0-362.8.1.el9_3.*
	kernel-core-0:5.14.0-362.8.1.el9_3.*
	```
## 버전 확인
- https://elrepo.org/linux/elrepo/el9/x86_64/RPMS/kmod-mpt3sas-43.100.00.00-6.el9_5.elrepo.x86_64.rpm
	- 위의 패키지는 sas2008 pci id 지원 확인
	- 문제는 9.5 iso 커널 버전이 다름
		- iso 버전: kernel-5.14.0-503.14.1.el9_5.x86_64.rpm
		- rpm 버전: 5.14.0-503.11.1.el9_5.x86_64
		```
		[root@localhost k]#  modinfo /root/lib/modules/5.14.0-503.11.1.el9_5.x86_64/extra/mpt3sas/mpt3sas.ko
		filename:       /root/lib/modules/5.14.0-503.11.1.el9_5.x86_64/extra/mpt3sas/mpt3sas.ko
		alias:          mpt2sas
		version:        43.100.00.00
		license:        GPL
		description:    LSI MPT Fusion SAS 3.0 Device Driver
		author:         Avago Technologies <MPT-FusionLinux.pdl@avagotech.com>
		rhelversion:    9.5
		srcversion:     2847634E42E03CCDF94C4B2
		alias:          pci:v00001000d000000E7sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E4sv*sd*bc*sc*i*
		alias:          pci:v0000117Cd000000E6sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E6sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E5sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000B2sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E3sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E0sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E2sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E1sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000D1sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000ACsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000ABsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000AAsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000AFsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000AEsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000ADsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C3sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C2sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C1sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C0sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C8sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C7sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C6sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C5sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C4sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C9sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000095sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000094sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000091sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000090sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000097sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000096sv*sd*bc*sc*i*
		alias:          pci:v00001000d0000007Esv*sd*bc*sc*i*
		alias:          pci:v00001000d000002B1sv*sd*bc*sc*i*
		alias:          pci:v00001000d000002B0sv*sd*bc*sc*i*
		alias:          pci:v00001000d0000006Esv*sd*bc*sc*i*
		alias:          pci:v00001000d00000087sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000086sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000085sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000084sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000083sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000082sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000081sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000080sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000065sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000064sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000077sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000076sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000074sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000072sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000070sv*sd*bc*sc*i*
		depends:        scsi_transport_sas,raid_class
		retpoline:      Y
		name:           mpt3sas
		vermagic:       5.14.0-503.11.1.el9_5.x86_64 SMP preempt mod_unload modversions
		sig_id:         PKCS#7
		signer:         ELRepo.org Secure Boot Key
		sig_key:        E9:D4:71:CF:B4:FE:13:6C
		sig_hashalgo:   sha256
		signature:      55:C9:D0:1B:38:AC:DB:97:B2:B5:88:DE:BF:C6:EC:8D:4C:26:53:2E:
		                5B:70:43:47:0F:91:13:70:9F:D0:90:16:EF:57:3B:5D:15:A4:55:02:
		                A9:D3:18:57:75:B1:42:7D:9F:5F:6D:8E:B9:9B:6B:1B:BE:3E:1F:C8:
		                16:49:ED:42:0B:B8:A7:AA:F1:9E:AC:07:30:CF:40:57:52:E0:EE:FE:
		                00:EC:C1:D1:0D:EC:81:53:2E:1C:D7:23:44:C7:C1:E6:D2:88:91:CC:
		                49:44:44:21:D4:5C:99:93:50:F3:67:1A:0E:A8:79:D8:26:68:22:D7:
		                B2:E1:9B:13:DE:59:28:8C:1B:E0:0E:5A:3A:A2:AE:FF:A1:8A:E0:45:
		                4E:93:FF:16:7B:DD:9B:CF:8A:14:29:EE:D3:4B:92:8C:DC:71:6F:C2:
		                8E:AA:A0:A2:9E:16:E3:D1:86:6F:EB:48:B7:36:B1:A8:06:9E:06:85:
		                40:28:E2:4D:A6:D5:62:91:49:AD:02:54:BE:12:ED:A2:8F:BB:FF:97:
		                78:8A:69:E3:99:E1:5B:DD:21:9D:46:15:FC:E7:A5:75:A3:8A:FD:F6:
		                EF:5A:6B:8C:C0:F9:32:97:32:6A:F6:D5:A0:ED:2F:DF:15:74:21:E9:
		                B3:90:19:CE:11:18:6D:F2:00:2D:05:F9:FA:7E:08:B8:E9:12:EE:5B:
		                92:7A:F8:12:9C:5C:98:6C:5D:3D:51:13:4B:92:3F:1B:9C:39:9B:A4:
		                E1:91:EB:4B:93:FE:63:47:99:D7:23:F2:E4:7E:E8:79:7D:8C:1C:0A:
		                0B:B2:8D:9C:EB:58:FC:FB:DB:30:B6:55:16:1E:DA:EE:12:77:F2:D6:
		                C2:C6:58:CA:CF:2C:CA:64:65:2D:2A:4B:F8:3D:25:9E:50:7A:91:CC:
		                BB:17:BA:29:DA:2C:95:12:DD:90:99:6C:AD:93:92:18:74:EC:AD:B1:
		                E3:59:40:FD:47:4F:60:8C:1D:23:5D:33:48:3B:86:02:97:68:59:D6:
		                23:1F:74:A4:1D:CA:D3:8D:92:5B:B7:42:CE:0B:B5:42:8A:66:9D:08:
		                0A:C7:37:A3:DE:97:00:E5:7F:40:EA:ED:CB:C0:CC:37:99:82:D6:43:
		                D5:08:11:48:93:AB:43:DD:C9:C1:A9:0A:D4:0B:A9:87:E8:D8:3B:3C:
		                84:2C:E4:90:05:8C:BF:1D:BE:A7:B5:09:FC:50:B3:59:D3:B7:87:C2:
		                45:B4:89:5A:7C:6C:1D:2F:D5:62:91:E2:18:02:A0:DC:CF:9C:E0:94:
		                AF:57:79:72:2C:6D:65:C6:04:CF:90:07:39:19:87:F4:35:F6:A5:3C:
		                7C:B6:5A:CF:24:CA:97:FB:9C:DF:91:C1
		```
- rocky linx v9.6 (다운로드)
	- https://mirror.navercorp.com/rocky/9.6/BaseOS/x86_64/os/Packages/k/
	- 마찬가지로 kmod rpm과 커널 버전 다름
- rocky 9.3
	- iso: 사이트 상으로는 버전 동일
	- kmod rpm: 5.14.0-362.8.1.el9_3.x86_64
		- https://elrepo.org/linux/elrepo/el9/x86_64/RPMS/ (dnf download)
		```
		[root@localhost mpt3sas]# modinfo mpt3sas.ko
		filename:       /home/test/lib/modules/5.14.0-362.8.1.el9_3.x86_64/extra/mpt3sas/mpt3sas.ko
		alias:          mpt2sas
		version:        43.100.00.00
		license:        GPL
		description:    LSI MPT Fusion SAS 3.0 Device Driver
		author:         Avago Technologies <MPT-FusionLinux.pdl@avagotech.com>
		rhelversion:    9.3
		srcversion:     683778234F3336E54173869
		alias:          pci:v00001000d000000E7sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E4sv*sd*bc*sc*i*
		alias:          pci:v0000117Cd000000E6sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E6sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E5sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000B2sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E3sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E0sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E2sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000E1sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000D1sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000ACsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000ABsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000AAsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000AFsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000AEsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000ADsv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C3sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C2sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C1sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C0sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C8sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C7sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C6sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C5sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C4sv*sd*bc*sc*i*
		alias:          pci:v00001000d000000C9sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000095sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000094sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000091sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000090sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000097sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000096sv*sd*bc*sc*i*
		alias:          pci:v00001000d0000007Esv*sd*bc*sc*i*
		alias:          pci:v00001000d000002B1sv*sd*bc*sc*i*
		alias:          pci:v00001000d000002B0sv*sd*bc*sc*i*
		alias:          pci:v00001000d0000006Esv*sd*bc*sc*i*
		alias:          pci:v00001000d00000087sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000086sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000085sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000084sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000083sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000082sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000081sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000080sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000065sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000064sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000077sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000076sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000074sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000072sv*sd*bc*sc*i*
		alias:          pci:v00001000d00000070sv*sd*bc*sc*i*
		depends:        scsi_transport_sas,raid_class
		
		```

