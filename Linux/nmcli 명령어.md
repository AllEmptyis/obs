## 기본 상태 확인
- nmcli
- nmcli device status
	- 인터페이스 상태 조회
- nmcli connection show `인터페이스명`
	- 연결 목록 조회
- nmcli general status
	- 전체 네트워크 상태 확인
## 인터페이스 활성화 / 비활성화
- nmcli device `connect / disconnect` eno1
- nmcli connection `up / down` br0
	- 연결 활성화
- nmcli connection add `type` `<type>` ifname `<연결할 인터페이스명>` con-name `<연결이름>` `[option]` 
	- 연결 생성
	- type
		- ethernet, bridge, bridge-slave, vlan, bond 등
			- 네트워크 매니저가 지원하는 연결 유형
	- option
		- master `<브릿지명>`
- nmcli connection delete `인터페이스명`
## IP 설정
- nmcli connection modify br0 ipv4.method `auto/manual`
- nmcli connection modify br0 ipv4.addresses 192.168.0.100/24
- nmcli connection modify br0 ipv4.gateway 192.168.0.1
- nmcli connection modify br0 ipv4.dns 8.8.8.8