# Docker
## 도커
- 도커란?
	- [[컨테이너|컨테이너]]를 쉽게 만들고 실행하기 위한 도구
		- 컨테이너는 리눅스의 cgroups, 네임스페이스 등의 기능을 사용하여 격리 된 환경
	- 컨테이너를 이미지로 만들고 실행 관리
		- 애플리케이션 환경 패키징 (dockerfile 사용)
- 실행 환경
	- 리눅스 전용
- Docker Desktop
	- 윈도우 / macOS 전용
	- 도커+가상환경 자동 구성 (GUI 제공)
	- 호스트와 자동으로 볼륨 연동 / 포트포워딩 설정
		- 호스트에 곧바로 바인드 마운트, 컨테이너 접속 가능
## Dockerfile
- 보통 컨테이너 이미지는 도커 허브에서 다운로드
- 커스텀 이미지를 만들기 위한 스크립트 파일
	- ex) ubuntu + nginx 설치 + 설정 파일
	- docker build 명령어로 하나의 이미지로 패키징 가능
## Docker Compose
- 여러 개의 컨테이너를 연결하여 실행하는 도구
	- 웹 + DB 등 같이 실행할 때
## 컨테이너 실행 시 알아둘 사항
- 이미지로 컨테이너 실행 시 이미지에 따라 실행해야 하는 커맨드가 있음
	- 도커 컨테이너는 내가 지정한 프로그램(프로세스)만 실행되기 때문
		- VM의 경우 OS 전체가 부팅 -> 자동으로 쉘 실행(tty)
		- 컨테이너는 내부에서 명시적으로 커맨드 실행 필요
			- ex) docker run ubuntu `bash`
			- 이미지 내부에 지정되어 있는 CMD만 실행
- `docker run ubuntu bash`
	- 우분투 이미지 안에서 bash 프로세스를 실행하라는 의미
	- 우분투 이미지 안에는 최소한의 실행 환경만 포함, 터미널 접속 위해서는 수동 커맨드 입력 필요
		- 프로세스를 실행하기 위한 파일 시스템만을 포함
# Docker 실습
## 실습 로드맵
- [x] 기본 패키지 설치
- [x] 컨테이너 띄우기
- [ ] 이미지 확인 (pull/tag/rmi & 캐시 이해)
	- 레이어 구조, 저장 위치 확인
- [ ] 볼륨 & 바인드 마운트
	- named volume vs bind mount 차이
- [ ] 네트워크 (bridge 사용자 정의, 포트 mapping)
	- iptables/nftables 규칙 확인
- [ ] Dockerfile 작성, 멀티스테이지 빌드
	- 이미지 빌드 원리, 캐시
- [ ] Compose로 다중 컨테이너 생성 (웹+DB)
	- 서비스 간 네트워크, env 전달, 스케일링
- [ ] 레지스트리 / 배포
	- 도커 허브 사용해서 pull/push
- [ ] 리소스 제약
## 기본 패키지 설치
- 도커 공식 저장소 추가
	- `dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`
- 관련 패키지 설치
	- `dnf install -y docker-ce docker-ce-cli containerd.io`
		- docker-ce: 도커 커뮤니케이션 에디션 (도커 무료 오픈소스 버전)
		- docker-ce-cli: 도커 명령어 cli 도구
		- containerd.io: 도커가 내부적으로 사용하는 컨테이너 런타임 (컨테이너 생성,관리)
```
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io
systemctl enable --now docker

[root@localhost ~]# docker --version
Docker version 28.3.2, build 578ccf6
```
## 컨테이너 띄우기
- 이미지 다운로드 시 로컬에 없으면 도커 허브에서 최신 이미지를 다운로드 (기본 경로: `docker.io/library/`)
	- 다른 레지스트리 지정 가능
- `docker run -d --name nginx -p 8080:80 nginx`
	- -d : 백그라운드에서 실행
	- --name: 컨테이너명
	- -p: 컨테이너의 80번 포트를 호스트의 8080으로 연결 (호스트 포트: 컨테이너 포트)
	- nginx: 이미지명
- `docker run -d --name [mariadb] -e [MYSQL_ROOT_PASSWORD=1234] -p [13306:3306] [mariadb:10.6]`
	- -e: 환경 변수 설정 (비밀번호)
	- mariadb:10.6 : tag, 10.6 버전으로 다운받겠다는 의미. 미지정 시 latest로 다운로드
- `docker exec -it [mariadb] [mysql -u root -p]`
	- mariadb 컨테이너 내부에서 mysql -u root -p로 접속
- `mysql -h [192.168.211.101] -P [13306] -u [root] -p`
	- 외부에서 mysql 접속 방법
	- -h: 접속할 도커 호스트의 ip, -P: 컨테이너의 포트
	- `firewall-cmd --list-all`
		- 접속 할 호스트에서 포트 열었는지 확인
		- `firewall-cmd --add-port=13306/tcp --permanent`
```
[root@localhost ~]# docker run -d --name nginx -p 8080:80 nginx
Unable to find image 'nginx:latest' locally   ----> nginx의 저장소 경로
latest: Pulling from library/nginx
59e22667830b: Pull complete
140da4f89dcb: Pull complete
96e47e70491e: Pull complete
2ef442a3816e: Pull complete
4b1e45a9989f: Pull complete
1d9f51194194: Pull complete
f30ffbee4c54: Pull complete
Digest: sha256:84ec966e61a8c7846f509da7eb081c55c1d56817448728924a87ab32f12a72fb
Status: Downloaded newer image for nginx:latest
6848dd5739f67ee5e4464d17c6201a80ac19f95530ca3dbd9dde5185fac5db04

//포트 확인
[root@localhost ~]# ss -ntlp | grep 13306
LISTEN 0      4096           0.0.0.0:13306      0.0.0.0:*    users:(("docker-proxy",pid=37529,fd=7))
LISTEN 0      4096              [::]:13306         [::]:*    users:(("docker-proxy",pid=37535,fd=7))

[root@localhost ~]# docker port mariadb
3306/tcp -> 0.0.0.0:13306
3306/tcp -> [::]:13306
```
## 이미지 레이어
- `docker history <img>`
	- image: 최상위 이미지는 id가 보이며, 하위 레이어는 `<missing>`으로 표시
	- created by: dockerfile 내 사용된 명령어
	- size: 해당 명령어로 추가된 파일 시스템 크기
	- comment: buildkit or 빌드 도구 관련 주석
- `docker image inspect <img>`

