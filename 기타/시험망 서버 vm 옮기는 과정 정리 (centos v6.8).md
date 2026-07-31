# 시험망 스위치 구성
- 시험망 pc 자체는 외부로 연결할 수 있는 인터페이스가 없음
- em1에 각각 ip 할당
	- 시험망1: 192.168.211.125
	- 시험망2: 192.168.211.126
- em1에서 남은 포트끼리 서로 연결해서 scp로 옮김
	- scp `파일명` root@ip:`옮길 경로`
# virtualbox vm import/export
- 가상머신 import 한 뒤 해당 ova 파일을 옮겨와서 export 하면 된다
- export 할 때 mac address policy(기존 mac addr 처리 방법)은 Generate new MAC addresses 선택
# VM 인터페이스 꼬임
- 기존 auto-ethx 인터페이스 지우고 ip a 했을 때 나오는 인터페이스로 설정파일 생성
- 해당 설정파일에 uuid,mac 주소 등 맞춰서 설정
- 경로
	- `/etc/sysconfig/network-scripts/`
- uuid 생성 방법
	- `uuidgen eth6`
- efcfg-ethx
	- onboot=yes, system name 변경 필요
- service network restart