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
```
도커는 리눅스 커널이 제공하는 쉘 위에서 동작. 즉 쉘 인터페이스를 통해 커널로 명령어를 전달하게 된다
쉘 종류:
bash, sh, ksh 등

컨테이너에 접속하는 방법:
docker exec -it [컨테이너 id] [/bin/bash]
-it : 인터렉티브 모드, 가상 터미널 오픈
/bin/bash : 컨테이너 내부의 bash 쉘 실행
다른 쉘로 실행하려면 컨테이너 안에 다른 쉘이 설치되어 있어야 함. (추가: alphine은 sh만 존재)

exec: 컨테이너 안에서 새로운 프로세스를 실행하라는 명령
docker run or start 이후 실행

docker run은 새 컨테이너 실행 명령어
```
# Docker 실습
## 실습 로드맵
- [x] 기본 패키지 설치
- [x] 컨테이너 띄우기
- [x] 이미지 확인 (pull/tag/rmi & 캐시 이해)
	- 레이어 구조, 저장 위치 확인
- [x] 볼륨 & 바인드 마운트
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
	- -p: 컨테이너의 80번 포트를 호스트의 8080으로 연결 **(호스트 포트: 컨테이너 포트)**
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
- 각 레이어는 도커파일의 명령어로 생성 됨
- `docker history <img>`
	- 이미지 레이어의 생성 과정 볼 수 있음
	- image: 최상위 이미지는 id가 보이며, 하위 레이어는 `<missing>`으로 표시
	- created by: dockerfile 내 사용된 명령어
	- size: 해당 명령어로 추가된 파일 시스템 크기
	- comment: buildkit or 빌드 도구 관련 주석
- `docker image inspect <img>`
	- 실제 레이어 디렉토리 확인 가능
		- lowerdir + upperdir = Mergeddir
- **파일 구조 설명**
	- upperdir(diff): 컨테이너의 쓰기 가능 계층 (RW layter)
		- 컨테이너에서 파일 생성,수정 시 해당 레이어에 기록 됨, 컨테이너 실행 시에만 생긴다
		- 혹은 실행 하면서 필수적으로 변경되는 것들이 생기기도 함
	- lowerdir: 이미지 레이어. 읽기 전용. docker image ls로 조회 가능
	- merged: 컨테이너 내부에서 보이는 파일 시스템
	- work: overlayFS의 작업용 임시 디렉터리
### 레이어 확인
```
//아래의 이미지들은 lower dir
[root@localhost mail]# docker history mariadb:10.6
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
c14f2faa3568   2 months ago   CMD ["mariadbd"]                                0B        buildkit.dockerfile.v0  --->최상위 이미지 레이어
<missing>      2 months ago   EXPOSE map[3306/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      2 months ago   ENTRYPOINT ["docker-entrypoint.sh"]             0B        buildkit.dockerfile.v0
<missing>      2 months ago   COPY docker-entrypoint.sh /usr/local/bin/ # …   26kB      buildkit.dockerfile.v0
<missing>      2 months ago   COPY healthcheck.sh /usr/local/bin/healthche…   10.3kB    buildkit.dockerfile.v0
<missing>      2 months ago   VOLUME [/var/lib/mysql]                         0B        buildkit.dockerfile.v0
<missing>      2 months ago   RUN |4 GPG_KEYS=177F4010FE56CA3336300305F165…   305MB     buildkit.dockerfile.v0
<missing>      2 months ago   RUN |4 GPG_KEYS=177F4010FE56CA3336300305F165…   134B      buildkit.dockerfile.v0
<missing>      2 months ago   ARG REPOSITORY=http://archive.mariadb.org/ma…   0B        buildkit.dockerfile.v0
<missing>      2 months ago   ENV MARIADB_VERSION=1:10.6.22+maria~ubu2004     0B        buildkit.dockerfile.v0
<missing>      2 months ago   ARG MARIADB_VERSION=1:10.6.22+maria~ubu2004     0B        buildkit.dockerfile.v0
<missing>      2 months ago   ENV MARIADB_MAJOR=10.6                          0B        buildkit.dockerfile.v0
<missing>      2 months ago   ARG MARIADB_MAJOR=10.6                          0B        buildkit.dockerfile.v0
<missing>      2 months ago   LABEL org.opencontainers.image.authors=Maria…   0B        buildkit.dockerfile.v0
<missing>      2 months ago   ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
<missing>      2 months ago   RUN |1 GPG_KEYS=177F4010FE56CA3336300305F165…   0B        buildkit.dockerfile.v0
<missing>      2 months ago   RUN |1 GPG_KEYS=177F4010FE56CA3336300305F165…   18.1MB    buildkit.dockerfile.v0
<missing>      2 months ago   ARG GPG_KEYS=177F4010FE56CA3336300305F1656F2…   0B        buildkit.dockerfile.v0
<missing>      2 months ago   ENV GOSU_VERSION=1.17                           0B        buildkit.dockerfile.v0
<missing>      2 months ago   RUN /bin/sh -c groupadd -r mysql && useradd …   329kB     buildkit.dockerfile.v0
<missing>      3 months ago   /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>      3 months ago   /bin/sh -c #(nop) ADD file:f9ee450324e6ff2c9…   72.8MB
<missing>      3 months ago   /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B
<missing>      3 months ago   /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B
<missing>      3 months ago   /bin/sh -c #(nop)  ARG LAUNCHPAD_BUILD_ARCH     0B
<missing>      3 months ago   /bin/sh -c #(nop)  ARG RELEASE                  0B

[root@localhost 1cf77d7492a8ebef6ea0536393a9fd7a83a4041d4eb4765392b2d232adf92adb]# ls -al
total 12
drwx--x---.  4 root root   72 Jul 28 02:57 .
drwx--x---. 22 root root 4096 Jul 28 03:13 ..
-rw-------.  1 root root    0 Jul 28 02:57 committed //해당 이미지 레이어가 커밋되었음을 표시하는 파일
drwxr-xr-x.  3 root root   33 Jul 28 02:57 diff    //upperlayer, 컨테이너 실행 후 변경 된 부분
-rw-r--r--.  1 root root   26 Jul 28 02:57 link // 레이어 참조용 링크 이름 저장
-rw-r--r--.  1 root root  115 Jul 28 02:57 lower //하위 레이어 경로 기록
drwx------.  2 root root    6 Jul 28 02:57 work
```
- 레이어 디렉토리 구조
```
[root@localhost ~]# cd /var/lib/docker/overlay2/
//이미지 레이어 나옴

[root@localhost overlay2]# ls -l l
total 0
lrwxrwxrwx. 1 root root 72 Jul 28 03:13 5WMWFXDVNHX7K3EV4MIDSSAOLK -> ../bcb1c132299e8668a097a96e3f3c88801e553c07857b0729b3d3aee0d606b270/diff
lrwxrwxrwx. 1 root root 72 Jul 28 02:57 7XM7L6ZEWYG7YAC5LFQH77JXFX -> ../66aa8e1a44f740345dd9f1dddf6f67c5e3fa2886d7c6592e158dbcab232e8981/diff
lrwxrwxrwx. 1 root root 72 Jul 28 03:13 A2J5NB6E7CDRTLCJXUZN327S5X -> ../4fef8fe92b32f1b69d7bfe3adf1c89433784ecdc0560c2a63fde53120a052ec5/diff
lrwxrwxrwx. 1 root root 72 Jul 28 02:57 AQFIGAYIECRSCKS62IO2DG3GTQ -> ../6cf7f281efecfd1f7a5dde7a83ea2633143a6f7c814a1da038d746c05e58fbfe/diff
lrwxrwxrwx. 1 root root 72 Jul 28 02:57 CMQKI56PMN25SYMXJY2FVPPYM3 -> ../390d37b02f8cbdb82f40f13bfe02cbac525a87bfc9cb815d2f4b71207796ab63/diff
lrwxrwxrwx. 1 root root 72 Jul 28 03:13 ETY7MYHZSRFU555N6C577PISH4 -> ../4b22cf007186b6cb1c6a30348c0da7c1b6b758fb34df62577a7af0fa7a06f30c/diff
lrwxrwxrwx. 1 root root 72 Jul 28 03:13 FT53HEYKW4MXBERAR7QC7DESHQ -> ../64dc7ca87ada4582f2702d689fe8620bd5f1df6b108730a37791a185b1d83025/diff
lrwxrwxrwx. 1 root root 77 Jul 28 03:13 GGR4G32I4LKLTT6F2RMZJMVYVS -> ../c852d77f3671efd675d9a630e3ec78e4dbd86eec1ad9a74d254b9fca9335611e-init/diff

GGR4G32I4LKLTT6F2RMZJMVYVS: 짧은 링크 이름
우측: 실제 경로
overlayFS가 lower파일에서 짧은 링크 이름을 참조해서 lowerdir을 구성

[root@localhost d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2]# cat lower
l/KGSQRTAMGX6R2I5J2RLMSZZXRC:l/TWI4QPPXDECM4NSQXC7BO57R4J:l/AQFIGAYIECRSCKS62IO2DG3GTQ:l/WWJ6TNXN7Y67OMSOU75PDKLWJS:l/CMQKI56PMN25SYMXJY2FVPPYM3:l/RAM4F6GXUTAPMIQV2FF24ORNRP:l/TSL5HAHT25W7HSMI7O365OEL73:l/7XM7L6ZEWYG7YAC5LFQH77JXFX[root@localhost d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2]#

최근 -> 베이스 순서
```
- docker inspect nginx로 이미지 레이어 경로 확인
```
        "GraphDriver": {
            "Data": {
                "ID": "6848dd5739f67ee5e4464d17c6201a80ac19f95530ca3dbd9dde5185fac5db04",
                "LowerDir": "/var/lib/docker/overlay2/d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2-init/diff:/var/lib/docker/overlay2/9b4f25e785d28150ccef405f7fed8b344b67d0641558dd7f48d1a11d503/diff:/var/lib/docker/overlay2/6cf7f281efecfd1f7a5dde7a83ea2633143a6f7c814a1da038d746c05e58fbfe/diff:/var/lib/docker/overlay2/1cf77d7492a8ebef6ea0536393a9fd7a83a4041d4eb4765392b2d232adf92adf:/var/lib/docker/overlay2/390d37b02f8cbdb82f40f13bfe02cbac525a87bfc9cb815d2f4b71207796ab63/diff:/var/lib/docker/overlay2/6ac0d05964c6a10b3b68b0b4750ea5ca4d61841064e09cc994acfe81d7bdc3cc/diff:/var/lib/r/overlay2/90647a47cccd36a8c728c10ed48fc900aec551018389f85e5151d4b9af632e21/diff:/var/lib/docker/overlay2/66aa8e1a44f740345dd9f1dddf6f67c5e3fa2886d7c6592e158dbcab232e8981/diff",
                "MergedDir": "/var/lib/docker/overlay2/d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2/merged",
                "UpperDir": "/var/lib/docker/overlay2/d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2/diff",
                "WorkDir": "/var/lib/docker/overlay2/d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2/work"
            },
            "Name": "overlay2"
```
- nginx merged 레이어 구조
```
[root@localhost 1cf77d7492a8ebef6ea0536393a9fd7a83a4041d4eb4765392b2d232adf92adb]# cd /var/lib/docker/overlay2/d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2
[root@localhost d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2]# ls -l
total 8
drwxr-xr-x. 5 root root  39 Jul 28 02:57 diff
-rw-r--r--. 1 root root  26 Jul 28 02:57 link
-rw-r--r--. 1 root root 231 Jul 28 02:57 lower
drwxr-xr-x. 1 root root  39 Jul 28 02:57 merged
drwx------. 3 root root  18 Jul 28 02:57 work
[root@localhost d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2]# ls diff //컨테이너 실행하면서 변경 된 부분
etc  run  var
[root@localhost d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2]# ls link
link
[root@localhost d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2]# ls merged //실제 컨테이너 들어갔을 때 보이는 구조
bin  boot  dev  docker-entrypoint.d  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
[root@localhost d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2]# ls work
work
```
#### 동작 확인
- 컨테이너에서 /etc/nginx/nginx.conf 수정
	- 원본은 lowerdir에 있음
	- diff/etc/nginx/nginx.conf로 복사 후 수정 됨 (Copy On Wirte)
	- merged는 diff와 lowerdir를 합쳐서 보여주는 것 (파일 시스템 가상 뷰)
```
[root@localhost d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2]# ls diff 
etc  run  var

[root@localhost d2cbf9d7384599bbc8ca004d2de13be83e8281782e6b5eaf7b6754901be9daa2]# ls -l diff/etc/nginx
total 4
drwxr-xr-x. 2 root root  26 Jul 28 02:57 conf.d
-rw-r--r--. 1 root root 644 Aug  5 01:26 nginx.conf

//변경분 생긴 것 확인
```
- 컨테이너 내부에서 파일 수정 방법
	- apt install vim 필요 or 호스트에서 컨테이너로 복사
- `docker exec -it nginx bash`
### pull/tag/rmi 및 캐시
- `docker pull`
	- `docker pull mariadb:10.6`
	- 도커 허브(원격 레지스트리)에서 이미지를 로컬로 다운로드
	- 동작 원리
		- 태그 확인 (ex.10.6) 후 레지스트리에서 매칭된 이미지 찾기
		- 이미지의 레이어 단위로 다운로드 / 만일 로컬에 있는 레이어라면 캐시 재사용 (중복 다운로드x)
	- 캐시란?
		- 도커 이미지 레이어를 재사용하는 매커니즘
		- 레이어 단위 SHA256 해시로 관리
		- 도커파일의 명령어, 관련 파일 해시 등을 비교하여 sha256해시 계산
		- docker pull로 이미지를 받으면 sha256해시 digest를 내려준다
			- 그 결과를 비교하여 현재 동일한 sha256값이 있으면 저장X (캐시 재사용)
- `docker tag`
	- `docker tag mariadb:10.6 mariadb:test`
	- 같은 이미지에 별칭 붙여서 새로운 컨테이너 생성 (이미지 레이어는 공유) / 메타데이터만 추가, 같은 이미지를 참조
		- 백업, 버전 관리, 배포 등에 활용
	- 같은 이미지 레이어를 공유하지만 수정하면 각 컨테이너의 upper dir에 새로운 데이터가 쓰여짐
		- docker commit하여 저장하면 해당 upper dir이 이미지 레이어가 된다
```
        [컨테이너 A]         [컨테이너 B]
          (writable)           (writable)
              │                    │
      ┌───────────────────────────────┐
      │  Image Layer N (CMD ...)      │
      │  Image Layer N-1 (COPY ...)   │
      │  ...                          │
      │  Base Image Layer             │
      └───────────────────────────────┘
```
- `docker rmi`
	- 로컬의 도커 이미지 제거
		- 태그, 레이어 참조를 관리
	- 태그 된 이미지가 있으면 해당 태그만 삭제하고, 참조가 더 이상 없으면 레이어까지 전부 삭제
## volume
- 컨테이너와 호스트 간 데이터 공유, 저장하기 위한 기능
	- 컨테이너는 종료하면 데이터도 같이 삭제 됨
- 볼륨 종류
	- named volume: **도커가 관리하는 디렉터리**
		- 기본경로: `/var/lib/docker/volumes/_data`
		- 내부에서는 볼륨을 생성한 위치로 보이고, 호스트에서 보면 위의 기본 경로에 볼륨이 저장 됨
		- 볼륨 생성 후 볼륨의 내부가 비어있으면 데이터 자동 복사
			- **마운트 할 볼륨 경로에 데이터가 있다면 볼륨의 데이터를 우선 사용**
	- bind mount: **호스트의 특정 디렉터리를 컨테이너와 공유**
	- tmpfs:프로세스 종료 시 데이터 삭제
- 명령어 (named)
	- `docker volume create <volume-name>` //볼륨 수동 생성
	- `docker run -v <볼륨이름>:<컨테이너경로> <container name> [:rw/ro]`
- 명령어 (bind)
	- `docker run -v <호스트경로>:<컨테이너경로> <container name> [:rw/ro]`
- 볼륨은 "컨테이너 생성 시" 같이 생성해야 함
	- 호스트와 공유하는 볼륨이 있는 파일시스템으로 만들어지는 것
- 비슷한 옵션 -  `--mount`
- 하나의 볼륨을 동시에 여러 컨테이너와 공유하는 방법
	- 호스트와 컨테이너 간 공유: 바인드
	- 컨테이너 간 공유: named 방식이 유리
### 실습 / nginx 컨테이너에서 html 파일 수정
- /usr/share/nginx/html 기본 루트를 공유 볼륨으로 잡기
	- 바인드 마운트하면 루트에 있는 파일은 가려짐 (호스트 볼륨의 데이터 내용만 보임)
```
[root@localhost /]# docker run -itd --name nginx_volume -p 8081:80 -v /host_volume:/usr/share/nginx/html nginx
5c06db850f9736105a7565de0945d3589dbd5761545b72ce0fea1c4b252bc48f
[root@localhost /]# docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                                     NAMES
5c06db850f97   nginx     "/docker-entrypoint.…"   3 seconds ago   Up 3 seconds   0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   nginx_volume
d18c0646b1e5   nginx     "/docker-entrypoint.…"   44 hours ago    Up 44 hours    0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   nginx_test

->호스트에서 index.html 생성하면 그대로 출력 됨
->그 전에는 403 forbidden
```
- named 방식
	- bind방식과 달리 볼륨 내용이 가려지지 않고 호스트에서 수정 가능
```
[root@localhost /]# docker run -d --name nginx_volume2 -p 8082:80 -v named_vomue:/usr/share/nginx/html nginx

[root@localhost _data]# pwd
/var/lib/docker/volumes/named_vomue/_data
[root@localhost _data]# ls
50x.html  index.html
```
- 볼륨 조회 방법
	- `docker volume ls`
	- `docker volume inspect [볼륨 이름]`
```
[root@localhost ~]# docker volume ls
DRIVER    VOLUME NAME
local     named_vomue

[root@localhost ~]# docker volume inspect named_vomue
[
    {
        "CreatedAt": "2025-08-08T17:50:41+09:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/named_vomue/_data",
        "Name": "named_vomue",
        "Options": null,
        "Scope": "local"
    }
]
```
## bridge 네트워크
- 각 컨테이너 내부에는 가상 nic이 있고 가상 nic은 호스트의 가상 nic veth랑 페어링 되어 있다
	- veth가 호스트 ip로부터 NAPT(Network Address Port Translation)을 이용하여 컨테이너와 통신하도록 함
- 기본 브리지 ip 대역
	- `172.17.0.0/16`
	- 컨테이너 시작 ip 주소: `172.17.0.2`
- 커스텀 브리지 ip 대역
	- `172.18.0.0/16` 부터 순차적으로 할당
- 도커 네트워크 종류
	- bridge
		- iptables 기반으로 변환 주소 변환
		- 리눅스 커널에 내장된 netfilter 커널 모듈을 통해 필터링 되며, iptables는 해당 커널 모듈을 이용할 수 있게 해주는 user space 프로그램
		- iptables 규칙을 통해 네트워크 격리 (docker chain안에 룰 설정)
	- host
		- 컨테이너와 호스트 간 네트워크 격리가 없으며, 호스트의 포트를 이용하여 통신 가능
		- bridge네트워크 없음
	- none
		- 외부와 통신x
- 커스텀 네트워크는 dns 통신 가능, 일반 브리지 네트워크에서는 ip로만 통신
	- dns 통신하기 위해서는 --link 및 /etc/hosts에 설정 필요 (커스텀 브리지 만드는 것 권장)
- 커스텀 브리지는 네트워크별 ip범위, 게이트웨이, 서브넷 설정 가능하며 내부적으로 docker dns(127.0.0.11) 자동 활성화
```
[root@localhost ~]# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
efc588d50f8b   bridge    bridge    local
97aec1c00d63   host      host      local
63598a25259d   none      null      local
```
### iptables 확인
```
[root@localhost ~]# iptables -nL -t nat //nat 테이블의 체인 출력 
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL //목적지가 로컬 주소인 경우 도커 체인으로 보냄, 만일 도커체인에서 일치하는 조건 없으면 다시 리턴

Chain INPUT (policy ACCEPT)
target     prot opt source               destination // 없음. 해당 규칙은 주로 호스트가 사용

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL //목적지가 로컬주소인 경우 도커 체인으로 보냄, 루프백 주소는 제외. (호스트에서도 컨테이너 접속 허용)

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0 //출발지가 도커ip대역인 경우 외부로 나갈 때 SNAT

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:80
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8081 to:172.17.0.3:80
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8082 to:172.17.0.4:80
```
### 명령어
- 네트워크 조회
	- `docker network ls`
	- `docker network inspect <id/네트워크명>`
	- `docker network inspect bridge`
	- `docker inspect -f '{{.Name}} - {{.NetworkSettings.IPAddress}}' $(docker ps -q)`
		- 실행 중인 모든 컨테이너 이름/ip를 한 줄씩 출력
- 생성
	- `docker network create <네트워크명>`
		- 브리지 드라이버 사용한 커스텀 네트워크 생성
	- `docker network create --driver bridge --subnet 192.168.100.0/24 --gateway 192.168.100.1 <네트워크명>`
		- 서브넷, 게이트웨이 지정해서 생성
		- 드라이버 종류
			- bridge, host, none, overlay
- 연결/해제
	- `docker network connect <네트워크명> <컨테이너명>`
	- `docker network disconnect <네트워크명> <컨테이너명>`
	- `docker run -d --name xx --network xx nginx`
		- 컨테이너 생성하면서 네트워크 지정할 수 있음 (이 경우는 지정한 네트워크에만 연결 됨)
		- 컨테이너는 여러 네트워크에 동시에 붙을 수 있음
- 네트워크 삭제
	- `docker network rm <네트워크명>`
	- `docker network prune`
		- 사용하지 않는 네트워크 전체 삭제
		- 사용 중인 네트워크는 삭제 불가
- 기타 트러블슈팅
	- `docker exec -it <컨테이너명> cat /etc/resolv.conf`
	- `docker exec -it <컨테이너명> ip a`
	- `docker inspect <컨테이너명> | grep -A 10 "Networks"`
		- 연결 된 네트워크 확인
### 실습
- 커스텀 브리지 생성 및 통신 확인
```
[root@localhost ~]# docker network create test
[root@localhost ~]# docker run -itd --name alpine_2 --network test alpine
[root@localhost ~]# docker run -it --name alpine_1 --network test alpine

15: br-60abe44a3dbd: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether ca:aa:6d:c0:16:cb brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-60abe44a3dbd
       valid_lft forever preferred_lft forever
    inet6 fe80::c8aa:6dff:fec0:16cb/64 scope link
       valid_lft forever preferred_lft forever

[root@localhost ~]# docker exec -it alpine_1 sh
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 5e:eb:40:2a:bc:ff brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever

/ # ping 172.18.0.2
PING 172.18.0.2 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.063 ms
^C
--- 172.18.0.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.063/0.063/0.063 ms
/ #
/ # ping alpine_2
PING alpine_2 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.052 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.026 ms
```
- 커스텀 브리지 iptables 확인
```
[root@localhost ~]# iptables -L -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere            !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.18.0.0/16        anywhere
MASQUERADE  all  --  172.17.0.0/16        anywhere

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere
RETURN     all  --  anywhere             anywhere
DNAT       tcp  --  anywhere             anywhere             tcp dpt:webcache to:172.17.0.2:80
DNAT       tcp  --  anywhere             anywhere             tcp dpt:tproxy to:172.17.0.3:80
DNAT       tcp  --  anywhere             anywhere             tcp dpt:us-cli to:172.17.0.4:80

[root@localhost ~]# docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                                     NAMES
9e739c23e2ec   alpine    "/bin/sh"                10 minutes ago   Up 10 minutes                                             alpine_2
65497fd87fd7   alpine    "/bin/sh"                11 minutes ago   Up 4 minutes                                              alpine_1
38a207cca48e   nginx     "/docker-entrypoint.…"   24 hours ago     Up 24 hours     0.0.0.0:8082->80/tcp, [::]:8082->80/tcp   nignx_named
ebcc18d9b343   nginx     "/docker-entrypoint.…"   24 hours ago     Up 24 hours     0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   nginx_hostvolume
d18c0646b1e5   nginx     "/docker-entrypoint.…"   5 days ago       Up 5 days       0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   nginx_test
```



-----
## 참고
- https://tech.cloudmt.co.kr/2022/06/29/%EB%8F%84%EC%BB%A4%EC%99%80-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88%EC%9D%98-%EC%9D%B4%ED%95%B4-3-3-docker-image-dockerfile-docker-compose/