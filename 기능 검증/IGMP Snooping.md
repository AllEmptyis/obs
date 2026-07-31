# 멀티캐스팅
- 동일한 ip 주소를 사용하여 같은 그룹끼리 동일한 메시지, 패킷을 수신
- 1:N 통신
- 종류
	- PIM (Protocol-Independent Multicast) / L3
	- IGMP (L2)
- 사용 주소 범위 (IP)
	- `224.0.0.0 ~ 239.255.255.255` (일부 주소는 예약되어 있음)
```
224.0.0.0   ~ 224.0.0.255    → 링크 로컬 (라우터 제어용, 라우터 넘어 못감)
224.0.1.0   ~ 238.255.255.255 → 글로벌 멀티캐스트 (인터넷용, IANA 할당)
239.0.0.0   ~ 239.255.255.255 → 사설 멀티캐스트 (조직 내부용)
```
- 사용 주소 (MAC)
	- `01:00:5E` 로 시작
	- 나머지 뒷자리는 igmp group ip를 mac 변환하여 사용
- 장점, 특징
	- 하나의 패킷이 전달되어 필요에 따라 복제, 분할 되어 전송
		- 이 때의 데이터 패킷: 스트림
	- 스트림을 위해 하나의 세션만 있어도 됨
- 대체로 udp 사용 (IPTV, 스트리밍)
	- tcp는 대체로 연결 기반이라 부적합함
- 멀티캐스트 통신 시 사용하는 ip/mac
	- igmp group ip/멀티캐스트 mac으로 멀티캐스트, igmp 모두 사용
# IGMP (Internet Group Management Protocol)
- 여러 장치가 하나의 ip를 공유하여 동일한 데이터를 수신할 수 있도록 하는 프로토콜
- `ipv4`에서 사용하는 멀티캐스팅 프로토콜
	- 멀티캐스트 그룹 join 및 leave 수행
- icmp와 마찬가지로 IP에서 직접 작동
- igmp group ip는 멀티캐스팅 하는 서버나 애플리케이션에서 정하는 것
	- 스트리밍 서버 등에서 그룹 ip를 설정
		- 보통 ip:port 로 관리 / 포트넘버까지 겹치면 멀티캐스팅 충돌
	- L3/라우터에서 igmp 쿼리 전송
	- 호스트에서 join/leave/report 응답을 전송

```
요약
IGMP: L2구간에서 호스트가 멀티캐스트 그룹에 가입/탈퇴 할 때 사용하는 프로토콜
PIM: L3구간에서의 멀티캐스트 라우팅
```
## igmp 메시지
- membership query
	- 라우터에서 특정 그룹에 가입한 호스트를 확인 할 때
	- 그룹별/소스별 질의 가능 (v3)
		- General Query: 224.0.0.1 / 모든 그룹에 대해 확인하고 싶을 때
		- Group-Specific Query: igmp group ip / 호스트가 특정 그룹에 대해 leave 메시지를 보냈을 때 해당 그룹에 사용자가 남아있는지 확인하기 위함
- membership report
	- 호스트가 그룹에 가입하고자 할 때 보냄
	- 수신/제외할 송신자 목록을 함께 보낼 수 있음 (v3)
	- 239.1.1.1 (지정한 igmp group ip)
- Leave group
	- 호스트가 그룹을 탈퇴할 때 보냄
	- 224.0.0.2
		- 모든 라우터에게 보냄
### 메시지 동작 흐름
1. 라우터 -> 멤버십 쿼리 송신
	- 네트워크 내 활성 그룹 멤버를 주기적으로 확인
2. 호스트 -> 멤버십 리포트 응답
3. 호스트 -> leave group 송신
	- 그룹 탈퇴 시 라우터에게 알림
# IGMP Snooping
- 라우터와 호스트가 igmp 쿼리,리포트,leave 메시지를 중간에서 보고 (또는 스위치 자신이 직접 스누핑 쿼리를 전송하여) igmp 그룹 멤버를 미리 파악하여 이후 멤버가 있는 포트로만 포워딩
- 각 포트로 쿼리 메시지를 전송하여 그룹 멤버 현황을 미리 파악 (igmp snooping querier)
- 라우터가 igmp 쿼리 메시지를 전송하면 브로드캐스팅 하지 않고 미리 파악해 둔 그룹 멤버에게만 report 메시지를 전송
- MAC 기반으로 동작
- L2가 쿼리어가 되면 상위 L3는 igmp 쿼리 메시지를 전송하지 않음
	- 선출 방식: ip가 낮은 장비가 쿼리어가 된다
## 기능
- igmp snooping querier (igmp v2 이상부터 설정 가능)
- igmp Robustness Variable
	- 스누핑 쿼리어 메시지에 포함된 값
	- 네트워크 불안정으로 인해 호스트의 응답이 손실되는 것을 방지하기 위함
		- 해당 값 만큼 호스트가 메시지를 스위치에게 전송해준다
- IGMP Snooping Last Member Query
	- 호스트로부터 leave 메시지를 수신한 경우 해당 그룹에 속한 멤버가 남아있는지 확인 용으로 보내는 쿼리
	- 해당 메시지에 대한 응답이 없으면 멀티캐스트 그룹 삭제
- IGMP Fast-Leave 기능
	- 호스트로부터 leave 메시지를 받았을 때 별도의 확인 과정 없이 해당 호스트를 바로 삭제해주는 기능
- **멀티캐스트 라우터 포트 설정**
	- 같은 vlan내에서 igmp 쿼리어가 여러 대 있을 경우 멀티캐스트 데이터 중복 전송 가능
		- 따라서 이러한 경우 ip 주소가 가장 작은 장비가 쿼리어로 동작
			- 동작: 쿼리어가 된 장비가 나머지 장비들에게 igmp general query를 주기적으로 송신 (자신보다 ip주소가 작은 경우 그 장비는 쿼리를 보내지 않음)
	- 그런데 만일 이 때 **tifront 장비의 ip가 라우터보다 작은 경우 tifront에서 전송하는 igmp query 메시지를 라우터가 수신할 수 있음**
		- 해당 라우터는 티프론트를 다른 라우터 장비로 인식해서 트래픽을 중단할 가능성이 있음
		- 이러한 경우를 막기 위해 멀티캐스트 라우터 포트를 설정 (쿼리 메시지를 보내지 않도록)
- IGMP Snooping Proxy
	- 스위치가 호스트들의 igmp 메시지를 대신 처리하고 라우터에게는 요약된 정보를 보내는 기능
	- snooping querier 와의 차이
		- 쿼리어는 쿼리 메시지를 생성
		- 프록시는 igmp report/leave 메시지를 대신 처리
## 설정
# 실습 (L2구간)
- VLC 설치해서 멀티캐스팅 스트리밍 서버 구성
- 서버: 리눅스
- 클라이언트: 윈도우
## 멀티캐스팅 스트리밍 서버 구성
- 미니멀 버전인 경우 사전 구성
```
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb

sudo dnf install -y https://mirrors.rpmfusion.org/free/el/rpmfusion-free-release-9.noarch.rpm
sudo dnf install -y https://mirrors.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-9.noarch.rpm
```
- vlc 설치 / 확인
```
//vlc는 루트 계정으로는 동작 안함 (내부적으로 네트워크 소켓 등을 사용해서 보안적인 목적 때문)

dnf install vlc -y
dnf install vlc-core vlc -y //console vlc

[test@localhost ~]$ vlc --version
VLC media player 3.0.23 Vetinari (revision 3.0.23-2-0-g79128878dd)
VLC version 3.0.23 Vetinari (3.0.23-2-0-g79128878dd)
Compiled by mockbuild on buildhw-x86-10.rdu3.fedoraproject.org (Jan 19 2026 00:00:00)
Compiler: gcc version 11.5.0 20240719 (Red Hat 11.5.0-11) (GCC)
This program comes with NO WARRANTY, to the extent permitted by law.
You may redistribute it under the terms of the GNU General Public License;
see the file named COPYING for details.
Written by the VideoLAN team; see the AUTHORS file.
```
- 스트리밍 할 영상 다운로드
```
sudo wget https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp_linux -O /usr/local/bin/yt-dlp

sudo chmod a+rx /usr/local/bin/yt-dlp

[test@localhost ~]$ yt-dlp --version
2025.09.26

[test@localhost ~]$ yt-dlp https://youtu.be/eMd5dZBzpfY?si=AwNzb1TzUe9m4VtF

[test@localhost ~]$ file bassdemo.mp4
bassdemo.mp4: ISO Media, MP4 v2 [ISO 14496-14]
//포맷 확인
```
- 송출 명령어
	- 송출 중지할 때는 kill -9 프로세스 번호
```
[test@localhost ~]$ cvlc bassdemo.mp4 \
 --intf dummy \
 --sout '#rtp{dst=239.1.1.1,port=5000,mux=ts}' \
 --loop --miface=ens160 &
[1] 84852
[test@localhost ~]$ VLC media player 3.0.23 Vetinari (revision 3.0.23-2-0-g79128878dd)
[0000561f5b560ef0] pipewire audio output error: PipeWire context connection error: Host is down
[0000561f5b560ef0] vlcpulse audio output error: PulseAudio server connection failure: Connection refused
[0000561f5b600990] dbus interface error: Failed to connect to the D-Bus session daemon: Unable to autolaunch a dbus-daemon without a $DISPLAY for X11
[0000561f5b600990] main interface error: no suitable interface module
[0000561f5b550b20] main libvlc error: interface "dbus,none" initialization failed
[0000561f5b62cfa0] main interface error: no suitable interface module
[0000561f5b550b20] main libvlc error: interface "globalhotkeys,none" initialization failed
[0000561f5b62cfa0] dummy interface: using the dummy interface module...
^C
```
## 클라이언트
- vlc 플레이어가 모종의 이유로 tx 스트림을 dmux 하지 못하여 mpv 사용
	- https://github.com/shinchiro/mpv-winbuild-cmake/releases
```
mpv-x86_64-20260306-git-3b55bc9.7z
이 파일 다운로드 받고 mpv.exe 실행
```
- cmd에서 mpv가 설치된 경로로 이동 후 명령어 실행
	- 성공하면 알아서 재생된다
```
C:\Users\USER\Downloads\mpv-x86_64-20260306-git-3b55bc9>
mpv.exe udp://@239.1.1.1:5000?localaddr=192.168.212.237
```

## 스위치
- igmp 쿼리어만 활성화