기존 CentOS 기반에서 RockyLinux 기반  Ticontroller로 마이그레이션
## 마이그레이션 가이드
1. 버전 확인
	- 로키 버전: 1.1.4.x.x
	- 기존 -> 신규 버전 업데이트 시 버전 뒷 2자리 일치
		ex) 1.1.3.5.6-> 1.1.4.5.6
2. 설정 백업
	1) Mysql 백업
		- mysqldump -u `DB id` -p`DB pw` --all-databases > `백업파일명`.sql
	2) Application.properties 백업
		- 기존에 있던 application.properties를 신규 티콘으로 옮김
		- `/cproject/conf/application.properties` 
	3) `.config` 파일 백업
		- cat /cproject/scripts/.config
3. RockyLinux 컨트롤러 설치
	- 위의 `.config` 파일 참조하여 신규 설치 진행 (mysql, es cluster name 등 동일하게 입력)
4. 컨트롤러 설정 복구
	1) mysql 복구 (스위치/컨트롤러 설정 복구)
		- mysql -u`DB id` -p`DB pw` < `백업파일명`.sql
	2) application properties 복구
		- 신규 컨트롤러의 application.properties 파일 백업 후 기존에 백업해둔 properties 파일을 /cproject/conf 경로에 옮김
			- 기존 파일은 중복되지 않도록 파일명 변경
		- application.properties의 버전명 변경 (복사한)
			- `project.version=v1.1.4.5.5`
		- 파일 권한 수정
			- 복사해 온 거라 root / root 로 되어 있음
			- 나머지 파일과 동일하게 ticontroller /controller로 수정
				- chown ticontroller:controller application.properties
	3) Ticontroller, mysql, elasticsearch 재시작 (systemctl restart)
## CentOS7.9 minimal TiController 설치 (1.1.3.5.7)
- rpm 다운로드 링크
	- [https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.E_eocwxtQg-wg-2aETwXFg](https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.E_eocwxtQg-wg-2aETwXFg)
	- [https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.IR2c7q7mQiijKJwdhJEHIg](https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.IR2c7q7mQiijKJwdhJEHIg)
- centos7 EOL로 인해 base url이 변경 되어 못 찾는 문제
	- `curl -fsSL https://autoinstall.plesk.com/PSA_18.0.62/examiners/repository_check.sh | bash -s -- update >/dev/null`
		- https://jfbta.tistory.com/324
- USB 마운트 (RPM 파일)
	- yum -y install ntfs-3g
	- mount -t ntfs-3g /dev/sdb1 /mnt/usb
- 패키지 압축 해제 및 설치
	```
	cp /mnt/usb/ticontroller-v1.1.3.5.7.tar.gz /root
	cp /mnt/usb/ticontroller-v1.1.3.5.7-local-rpm.tar.gz /root
	
	tar -xvzf ticontroller-v1.1.3.5.7.tar.gz
	
	[root@localhost ~]# ls
	anaconda-ks.cfg  ticontroller-v1.1.3.5.7-local-rpm.tar.gz
	ticontroller     ticontroller-v1.1.3.5.7.tar.gz
	```
- 설치
	- `/ticontroller/scripts/install.sh`  실행
	- 정상적으로 설치 시, `/cproject` 디렉토리 생성
- 설정
	- `/cproject/scripts/all_config.sh 실행 후 설정 -> apply
		- 비번은 two**, 계정 id는 전부 test로 설정
	- `ticontroller_check.sh` 정상 작동 확인
	- `https://ip:8443`으로 접속
- 라이센스 입력 (사내망에서 발급)
- 초기 ip/pw
	- super / admin123!@#
### 백업
- mysql 백업
	```
	[root@localhost ~]# mysqldump -u test -p --all-databases > backup.sql
	Enter password:
	[root@localhost ~]# ls
	anaconda-ks.cfg  backup.sq
	```
- 백업 파일 usb에 복사
	- cp backup.sql /mnt/usb
	- cp application.properties /mnt/usb
		```
		[root@localhost conf]# ls /mnt/usb
		application.properties     ticontroller-v1.1.3.5.7-local-rpm.tar.gz  ticontroller-v1.1.4.5.7.tar.gz
		backup.sql           
		```
- cat /cproject/scripts/.config (컨트롤러 설정 파일 복사)
	```
	[root@localhost scripts]# cat .config
	ifname=ens33
	hostname=localhost.localdomain
	ipaddr=192.168.212.181
	netmask=255.255.255.0
	gateway=192.168.212.1
	dns1=192.168.203.100
	dns2=168.126.63.1
	timezone=Asia/Seoul
	rootpw=twoneulbo0510!
	userid=test
	userpw=twoneulbo0510!
	es_cluster_name=test
	heapsize=1
	configtype=config
	```
## Rocky Linux9.2 minimal 설치 (1.1.4.5.7)
- rpm 다운로드 링크
	- [https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.zTOEXZB4QiWndhR4BbhgWA](https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.zTOEXZB4QiWndhR4BbhgWA)
	- [https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.JXGDaeCVQl6kPjJVv4pTsA](https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.JXGDaeCVQl6kPjJVv4pTsA) (rpm)
- 설치
	- 소프트웨어 선택 - Rocky Linux 표준 설치 체크
	- 파티션 설정
		- /boot: 1024
		- /boot/efi: 1024
		- /
- usb 마운트
	- dnf install epel-release -y
	- dnf install -y ntfs-3g
- 패키지 설치
- 최초 설치 시, python 필수 모듈 설치 필요
	- `python install_pip.py`
- 설치 스크립트 실행
	- `/root/ticontroller/operator/run.py`
	- `python run.py`
- 의존성 문제 발생 (오류)
	- subprocess.CalledProcessError: Command 'dnf -y install /root/ticontroller/local_rpm/base_tools/pciutils_3.7.0/*.rpm --disablerepo=*' returned non-zero exit status 1.
```
Traceback (most recent call last):
  File "/root/ticontroller/operator/run.py", line 115, in <module>
    main()
  File "/root/ticontroller/operator/run.py", line 111, in main
    select_feature()
  File "/root/ticontroller/operator/run.py", line 86, in select_feature
    f_ticon.run()
  File "/root/ticontroller/operator/src/ticontroller/feature_ticon.py", line 66, in run
    ti_inst.install()
  File "/root/ticontroller/operator/src/ticontroller/install.py", line 315, in install
    self.install_base_tools()
  File "/root/ticontroller/operator/src/ticontroller/install.py", line 255, in install_base_tools
    self.sc.dnf_install(tool_dir, ssh)
  File "/root/ticontroller/operator/src/common/system_call.py", line 122, in dnf_install
    self.run_cmd(cmd, ssh)
  File "/root/ticontroller/operator/src/common/system_call.py", line 68, in run_cmd
    ret = self.local_run_cmd(cmd, mute)
  File "/root/ticontroller/operator/src/common/system_call.py", line 55, in local_run_cmd
    raise subprocess.CalledProcessError(r_code, cmd)
subprocess.CalledProcessError: Command 'dnf -y install /root/ticontroller/local_rpm/base_tools/pciutils_3.7.0/*.rpm --disablerepo=*' returned non-zero exit status 1.
```
- 해결
	- 엔지니어 포털에 있는 ISO 사용
	- dnf update X
	- rpm usb에 담지 않고 wget으로 직접 다운로드
	- 원인 무엇인지 확인해 봐야 할 거 같다
### 백업
- 백업 파일이 들어있는 usb 마운트 후 mysql 복구
	- mysql -u test -p < backup.sql
- application properties 복구
	```
	[root@localhost usb]# mv /cproject/conf/application.properties application.properties_backup
	[root@localhost usb]# cp application.properties /cproject/conf/
	[root@localhost usb]# ls /cproject/conf
	application.properties  logback-nvt.xml  product.properties  server.cnf        ticontroller.key
	license.properties      logback.xml      restore.properties  ticontroller.crt
	```
- 버전 변경
- 파일 권한 변경
	```
	[root@localhost conf]# ls -al
	total 48
	drwxr-xr-x.  2 ticontroller controller 4096 Jun 12 10:48 .
	drwxr-xr-x. 10 ticontroller controller  108 Jun 12 10:36 ..
	-rwxr-xr-x.  1 root         root       9949 Jun 12 10:48 application.properties
	-rw-r--r--.  1 ticontroller controller  136 Jun 12 10:36 license.properties
	-rw-r--r--.  1 ticontroller controller 2031 Jun 12 10:36 logback-nvt.xml
	-rw-r--r--.  1 ticontroller controller 2430 Jun 12 10:36 logback.xml
	-rw-r--r--.  1 ticontroller controller  141 Jun 12 10:36 product.properties
	-rw-r--r--.  1 ticontroller controller    6 Jun 12 10:36 restore.properties
	-rw-r--r--.  1 ticontroller controller  989 Jun 12 10:36 server.cnf
	-rw-r--r--.  1 ticontroller controller 1253 Jun 12 10:36 ticontroller.crt
	-rw-r--r--.  1 ticontroller controller 1853 Jun 12 10:36 ticontroller.key
	[root@localhost conf]#
	[root@localhost conf]# chown ticontroller:controller application.properties
	[root@localhost conf]#
	[root@localhost conf]# ls -al
	total 48
	drwxr-xr-x.  2 ticontroller controller 4096 Jun 12 10:48 .
	drwxr-xr-x. 10 ticontroller controller  108 Jun 12 10:36 ..
	-rwxr-xr-x.  1 ticontroller controller 9949 Jun 12 10:48 application.properties
	-rw-r--r--.  1 ticontroller controller  136 Jun 12 10:36 license.properties
	-rw-r--r--.  1 ticontroller controller 2031 Jun 12 10:36 logback-nvt.xml
	-rw-r--r--.  1 ticontroller controller 2430 Jun 12 10:36 logback.xml
	-rw-r--r--.  1 ticontroller controller  141 Jun 12 10:36 product.properties
	-rw-r--r--.  1 ticontroller controller    6 Jun 12 10:36 restore.properties
	-rw-r--r--.  1 ticontroller controller  989 Jun 12 10:36 server.cnf
	-rw-r--r--.  1 ticontroller controller 1253 Jun 12 10:36 ticontroller.crt
	-rw-r--r--.  1 ticontroller controller 1853 Jun 12 10:36 ticontroller.key
	```
- 데몬 재시작
	- systemctl restart `ticontroller, mysql, elasticsearch`
- 확인
	- 바뀐 ip로 티콘 접속해보면 기존 설정이 그대로 옮겨진 것 확인
## TiController에 스위치 연동
### ZTI 방식으로 연동하기
- 제로 터치 설치 (Zero Touch Installation)
	- 스위치가 없어도 웹(원격)에서 미리 스위치 설정 가능 - 가상 스위치 설정 기능 
	- 이후 컨트롤러에 접속해서 스위치 전원/인터넷만 연결 되면 설치 가능
