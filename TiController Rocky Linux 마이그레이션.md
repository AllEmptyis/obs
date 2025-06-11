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
		- application.properties의 버전명 변경
			- `project.version=v1.1.4.5.5`
		- 파일 권한 수정
	3) Ticontroller, mysql, elasticsearch 재시작
## CentOS7.9 minimal TiController 설치 (1.1.3.5.7)
- rpm 다운로드 링크
	- [https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.E_eocwxtQg-wg-2aETwXFg](https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.E_eocwxtQg-wg-2aETwXFg)
	-  [https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.IR2c7q7mQiijKJwdhJEHIg](https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.IR2c7q7mQiijKJwdhJEHIg)
- EOL로 인해 레포지토리 사용 불가 / usb로 옮김
## Rocky Linux9.2 minimal 설치 (1.1.4.5.7)
- rpm 다운로드 링크
	- [https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.zTOEXZB4QiWndhR4BbhgWA](https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.zTOEXZB4QiWndhR4BbhgWA)
	- [https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.JXGDaeCVQl6kPjJVv4pTsA](https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.JXGDaeCVQl6kPjJVv4pTsA) (rpm)