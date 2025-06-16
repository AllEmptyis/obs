### IPTABLES
- 패킷 필터 시스템
- 커널 수준의 방화벽 엔진
### 구성
- Chain: 패킷을 언제 가로채서 검사할지 결정 (pre, post 등)
- Table: 패킷에 어떤 처리를 할지 결정
	- 기본: filter (패킷 허용/차단), nat(주소 변경)
- target: 규칙에 매칭 된 패킷을 어떻게 처리할지
	- accept, drop 등
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
#### TARGET
- 일반 타깃은 `filter` 테이블에서 사용
	- 테이블 명시 없어도 적용
- ACCEPT
	- 패킷 허용
- DROP
	- 폐기
- REJECT
	- 패킷 거부 + 오류 응답 전송(icmp 등)
- RETURN
	- 호출한 체인으로 되돌아감
- LOG
	- 패킷을 로그 기록 (`dmesg`, `/var/log/messages`)
- 사용자 정의 체인 이름도 타깃으로 사용 가능
##### 특수 타깃 (NAT)
- NAT 테이블에서만 사용 가능한 타깃
- MASQUERADE
	- sip를 호스트 동적 ip로 변환 / 주로 dhcp 환경에서 많이 사용
- SNAT
	- 출발지 ip를 고정 ip로 변환
- DNAT
	- DIP를 다른 주소로 변환 
	- 포트포워딩
- REDIRECT
	- 패킷이 들어올 때 DIP를 로컬 호스트로 redirection
	- 프록시 서버 (패킷이 들어오는 서버를 경유)

## IPTABLES 명령어
- Rocky Linux 9.x 기준 iptables가 아닌 `nftables`로 동작
- 호환성 때문에 iptables도 동작
	- `iptables-nft` 백엔드를 사용해서 내부적으로 연결
### 테이블 조회
- iptables `-L` `-n` `-v`
	- -L: 체인
	- -n: ?
	- -v: 패킷/바이트 수 표시
- iptables -t `nat` -L -v -n
	- `-t nat` : NAT 테이블 지정
	- NAT 테이블 조회
- iptables -L `INPUT` -v -n
  iptables -t `nat` -L `POSTROUTING` -v -n
	- 특정 체인만 조회
- iptables -L `--line-numbers`
	- 규칙 라인 번호까지 조회
- iptables-save
  /etc/sysconfig/iptables
	- 모든 테이블/체인/규칙 조회