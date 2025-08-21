# QoS (Quality of Service)
- 트래픽에 따라 대역폭을 제한할 수 있는 기능
	- 보통 동작은 수신한 패킷 부터 차례대로 전송
	- 대역폭이 부족하면 중요 트래픽이 늦게 전송될 수 있는 문제 발생
- 트래픽을 클래스 별로 분류 후 정책 or 최대 대역폭 적용
## 분류
- class (11개) - 패킷을 어떤 조건으로 분류할지? // 해당 조건에 만족하면 클래스로 분류 됨
	- 출발지/목적지 ip,mac,포트 번호
	- DSCP
	- 이더넷 타입
	- IP 프로토콜
	- 패킷을 수신한 인터페이스
	- VLAN
- 정책 - 클래스 별로 적용 될 대역폭 정의 (액션)
	- 클래스
	- 우선순위 - 같은 정책에 여러 클래스가 정의 된 경우 어떤 클래스를 우선 적용할지
	- 대역폭 (클래스에 해당하는 패킷/특정 포트 범위)
## 동작 원리
- 대역폭 제한 (rate-limit)
```
대역폭 처리: polishing, shaping
shaping: 설정한 대역폭 이상 들어오면 버퍼에 저장해두었다가 나중에 전송
polishing: 설정한 대역폭 이상 들어오면 드롭 처리하고, 버퍼에 새 트래픽을 받음

confirm-action에서는 폴리싱 방식 사용 (클래스에 보장되는 최대 대역폭, 최대 버스트)
servie-queue output rate-limit: shaping 방식 사용
(특정 포트에 적용되는 송수신 대역폭 제한)
```
- 큐 스케줄링
```
출력포트에서 큐에 저장된 패킷이 전송 가능한 대역폭보다 많은 경우 어떻게 처리할지

SPQ: 각 큐에 우선순위 정의, 우선순위가 높은 큐를 먼저 처리
RR: 큐를 순차적으로 선택
WRR: 큐마다 weight 값을 설정하고, 설정한 weight 비율만큼 패킷 처리
DRR: quantun, deficit counter 정의 후 deficit 카운터 크기만큼 큐의 패킷 처리
```
- 용어 정리
```
CoS: Class of service
L2트래픽 우선순위 표시 (VLAN태그에 들어있는 3비트 PCP필드가 CoS 값이다.)
3비트: 총 8개 우선순위
숫자 높을수록 우선순위 높음
vlan 태그 없는 일반 이더넷 프레임은 cos값 없음.

DSCP: L3 IP 헤더에 있는 값 (6비트: 64단계)
ipv4헤더에 있는 tos필드(현재는 거의 dscp+ecn으로 사용)
기본값: 0
->따라서 패킷이 장비로 들어오면 dscp값을 새로 넣어서 qos 설정을 할 수 있음

보통 CoS 값을 따라서 큐가 8개로 구성

트래픽이 들어오게 되면 Cos, dscp 값에 따라서 특정 큐로 매핑됨
->트래픽이 들어오면서 마킹 안한 경우 보통 0
->두 필드 다 마킹 된 경우 tifront는 dscp로 큐매핑 하는 듯
보통 L3장비:dscp기준
L2:cos기준

트래픽 들어옴 -> 클래스 기반으로 분류 > 특정 큐로 매핑 > 실제 큐에서 어떤 방식으로 처리할지 결정(스케쥴러)

큐 스케줄링 필요한 이유:
우선순위가 높은 클래스는 높은 우선순위 가진 큐로 매핑됨
그런데 우선순위 큐만 먼저 처리하다보면 다른 패킷은 처리되지 않을 수 있음
이런 문제를 해결하고자 큐에서 실제로 어떻게 처리할지 스케줄링 필요
```
## 설정 -CLI
- class map
	- qos
	- class-map `<map name>`
	- match `<분류 기준 정의>`
- policy map
	- qos
	- policy-map `<policy name>`
	- class `<적용할 클래스명> <우선순위>`
	- confirm-action `qos action`
		- **패킷 차단/허용 , dscp 값 삽입, 우선순위 설정 등**
	- rate-limit `<클래스의 트래픽에 보장해줄 최대 대역폭> <클래스의 트래픽이 사용할 수 있는 최대 버스트>`
		- 버스트: 순간적으로 몰리는 트래픽량 허용 범위
		- 최대 대역폭 설정보다 큰 트래픽이 들어오게 되면 허용된 버스트 만큼 잠시 허용
- service policy (적용)
	- service-policy `<정책 이름>`
	- 한 정책만 적용 가능
- 큐 스케줄링 설정
	- service-queue output `<ifname>` schedule mode `{drr/rr/spq/wrr}`
	- service-queue input `<ifname>` cos-map `{defualt/Cos우선순위/큐번호}`
		- CoS 필드의 우선순위에 따라 전송 큐 처리하도록 설정
- 송/수신 대역폭 제한 설정 (특정 포트)
	- service-queue output `<ifname>` rate-limit `{<최대대역폭 최대버스트>/none}`
- 정의한 큐에 대역폭 제한 설정
	- service-queue output `<ifname>` cos-rate-limit `<큐번호> {<최소 대역폭 최대 대역폭>/none}`
- 조회
	- show class-map / policy-map / service-policy
	- service-queue `input/output <ifname>`
## 설정 - Ticontroller
- ACL 설정에서 네트워크 접근제어 설정할 때 qos 설정 가능
	- dscp, cos, rate-limit 설정 가능
		- rate-limit만 설정한 경우 설정한 값에 맞게 트래픽 인가
		- dscp,cos와 함께 rate-limit 설정한 경우: dscp 값이 매칭 되는 트래픽에 대해 rate-limit 동작 ->왜 cos값은 안보는지?
# 실습
- 실습 도구: iperf3 (네트워크 성능 테스트 툴)
	- https://iperf.fr/iperf-download.php
- 설치 완료
	- 고급 시스템 설정 > path > iperf3 파일 있는 경로 추가 > cmd 관리자 권한 실행
```
C:\Windows\System32>cd C:\Users\USER\Desktop\Dabin\iperf3

C:\Users\USER\Desktop\Dabin\iperf3>iperf3 -v
iperf 3.19.1 (cJSON 1.7.15)
CYGWIN_NT-10.0-26100 000998-01 3.6.4-1.x86_64 2025-07-15 07:55 UTC x86_64
Optional features available: CPU affinity setting, support IPv4 don't fragment, POSIX threads
```