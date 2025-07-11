# Docker 명령어 및 옵션
## 컨테이너
- `docker run [옵션] [image명] [command] [arg]`
	- 컨테이너 생성 + 실행
	- **주요 옵션**
	    - `-d` : 백그라운드 실행
	    - `-p 호스트포트:컨테이너포트` : 포트 매핑
	    - `--name 이름` : 컨테이너 이름 지정
	    - `-it` : 터미널 연결(interactive + tty)
	    - `--rm` : 컨테이너 종료 시 자동 삭제
	    - `-v 호스트디렉토리:컨테이너디렉토리` : 볼륨 마운트
	- command
		- 컨테이너 내부에서 실행 할 명령

- `docker ps`
	- 현재 실행 중인 컨테이너 목록

- `docker ps -a`
	- 종료된 것 포함 모든 컨테이너 목록

- `docker [명령어] [컨테이너ID/이름]`
	- stop: 컨테이너 정지
	- start: 컨테이너 시작
	 - restart: 컨테이너 재시작
	 - rm: 컨테이너 삭제
  
- `docker exec -it [컨테이너ID/이름] bash`
	- 기존 컨테이너 내부 bash 쉘 접속

- `docker logs [컨테이너ID/이름]`
	- 컨테이너 로그 확인
## 이미지
- `docker images`
	- 로컬에 존재하는 이미지 목록 조회
- `docker pull [이미지명:태그]`
	- 이미지 다운로드
- `docker build -t 이미지명:태그 .`
	- dockerfile 기준으로 이미지 빌드
	- docker build -t myapp .
- `docker rmi [이미지ID/이름]`
	- 이미지 삭제
## 네트워크 / 볼륨 / 기타
- `docker network ls`
	- 네트워크 목록 조회
- `docker volume ls`
	- 볼륨 목록
- `docker info`
	- 도커 엔진 상태 조회 (네트워크 정보, 실행환경 등)
- `docker inspect [이미지명 or 컨테이너id]`
	- 도커 이미지/컨테이너의 내부 정보를 json 형식으로 보는 명령어
	- 이미지
		- cmd, entrypoint
	- 컨테이너
		- 어떤 이미지로부터 만들어졌는지
		- 현재 상태 (ruuning, exited)
		- 마운트 정보, 네트워크 설정 등
## 기타 옵션
- `--env KEY=VALUE`
	- 환경 변수
- `--network 네트워크명`
	- 네트워크 지정