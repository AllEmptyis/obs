### IPTABLES
- Chain: 패킷을 언제 가로채서 검사할지 결정 (pre, post 등)
- Table: 패킷에 어떤 처리를 할지 결정
	- 기본: filter (패킷 허용/차단), nat(주소 변경)
#### chain
- chain INPUT
	- 목적지가 '로컬 시스템'인 패킷이 도달하게 되는 체인 / 라우팅 된 후 도착
	- 로컬 접근 차단 / 허용
- chain prerouting
	- 패킷이 시스템에 들어올 때 **가장 먼저 거치는** 체인
	- 라우팅 전에 적용
		- 패킷 들어올 때 해당 ip로 라우팅 하지 않고 dnat 해주는 용도
	- DNAT에 사용
- chain postrouting
	- 패킷이 전송되기 전 마지막으로 거치는 체인
		- 라우팅 경로 결정 후, 최종적으로 패킷 전송될 때 SNAT 할 때 주로 사용
- chain OUTPUT
	- 패킷이 외부로 나갈 때 거치는 체인
		- 호스트에서 발생한 트래픽에 대한 방화벽
- FORWARD
	- 목적지가 다른 서버인 경우 (라우팅)
#### chain별 테이블
- 체인 별 주로 사용하는 테이블 / 용도
1) prerouting
	- nat, mangle, raw
	- DNAT, QoS
2) input
	- filter, mangle
	- 로컬 접근 제어
3) output
	- filter, nat, mangle
	- 로컬 발신 제어
4) postrouting
	- nat, mangle
	- SNAT, QoS
#### 테이블
- 사용자가 테이블 지정 시에만 동작한다
- filter
	- 기본 방화벽 기능 (차단/허용)
- mangle
	- 패킷 세부 속성 설정 (헤더 등)
	- TTL, TOS(QoS) 등
- nat
	-  주소 변환 (SNAT/DNAT)
- raw
	- 커넥션 추적 제외, 초기 처리
- security
	- SELinux 보안 처리