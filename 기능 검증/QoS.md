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
	- https://redmine.piolink.com/projects/99/wiki/QOS_Policing_vs_Shaping?search_id=1755755720.8364954&search_n=0
```
대역폭 처리: polishing, shaping
shaping: 설정한 대역폭 이상 들어오면 버퍼에 저장해두었다가 나중에 전송
polishing: 설정한 대역폭 이상 들어오면 드롭 처리하고, 버퍼에 새 트래픽을 받음

confirm-action에서는 폴리싱 방식 사용 (클래스에 보장되는 최대 대역폭, 최대 버스트)
servie-queue output rate-limit: shaping 방식 사용 (특정 포트에 적용되는 송수신 대역폭 제한)
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
- QoS 알고리즘
```
원인:
토큰 버킷 개념으로 보면
CIR(보충속도): 초당 채워지는 토큰 양
    - 500 Mbps ⇒ 초당 500 Mb 만큼 토큰이 채워짐
    - 10 Mbps ⇒ 초당 10 Mb 만큼만 채워짐
CBS(버스트용량): 한 번에 꺼내 쓸 수 있는 토큰 최대치    
    - 두 경우 모두 10 Mb

udp 8mbps는 평균치이고
실제 송신 시에는 피크가 발생한다.

rate-limit이 500mpbs일 땐 순간 버스트가 올라도 토큰이 금방 채워짐
그러나 10mbps일 땐 토큰 충전이 느려서 버킷을 채우는 속도가 저하. 드롭 발생
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
	- 주의: rate-limit, confirm-action 동시에 설정 안 됨
- service policy (적용)
	- qos
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
	- show service-queue `input/output <ifname>`
## 설정 - Ticontroller
- ACL 설정에서 네트워크 접근제어 설정할 때 qos 설정 가능
	- dscp, cos, rate-limit 설정 가능
		- rate-limit만 설정한 경우 설정한 값에 맞게 트래픽 인가
		- dscp,cos와 함께 rate-limit 설정한 경우: dscp 값이 매칭 되는 트래픽에 대해 rate-limit 동작 ->왜 cos값은 안보는지?
# 실습 - cli
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
- iperf 사용법
	- 서버: iperf -s
	- 클라이언트: iperf3 -c `서버ip` -u -b `대역폭` -l `1200` -t `시간`
		- 패킷 크기 지정 반드시 필요->안하면 스위치에서 드롭함
### 대역폭 제한 (policing)
- 설정 오류 - confirm action permit과 rate limit 같이 설정 안 됨
```
TiFRONT(config-qos-pamp-class)# rate-limit 500000 1
% Action Conflict with Permit
```
- QoS 설정 없을 때
	- 손실률 0.9%
```
C:\Users\parkd\Desktop\case>iperf3 -c 10.10.10.10 -u -b 1000M -l 500 -t 10
Connecting to host 10.10.10.10, port 5201
[  5] local 10.10.10.20 port 51755 connected to 10.10.10.10 port 5201
[ ID] Interval           Transfer     Bitrate         Total Datagrams
[  5]   0.00-1.01   sec  38.9 MBytes   322 Mbits/sec  81681
[  5]   1.01-2.01   sec  38.5 MBytes   324 Mbits/sec  80801
[  5]   2.01-3.01   sec  38.4 MBytes   323 Mbits/sec  80512
[  5]   3.01-4.00   sec  38.5 MBytes   324 Mbits/sec  80835
[  5]   4.00-5.00   sec  38.7 MBytes   326 Mbits/sec  81141
[  5]   5.00-6.01   sec  38.8 MBytes   322 Mbits/sec  81316
[  5]   6.01-7.01   sec  38.3 MBytes   321 Mbits/sec  80363
[  5]   7.01-8.01   sec  38.5 MBytes   325 Mbits/sec  80769
[  5]   8.01-9.02   sec  39.0 MBytes   324 Mbits/sec  81767
[  5]   9.02-10.01  sec  38.2 MBytes   322 Mbits/sec  80091
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.01  sec   386 MBytes   323 Mbits/sec  0.000 ms  0/809276 (0%)  sender
[  5]   0.00-10.01  sec   382 MBytes   320 Mbits/sec  0.013 ms  7317/809276 (0.9%)  receiver

iperf Done.
```
- 정책, 클래스 설정
	- 최대 대역폭: 500mbps
	- 버스트: 10mbps
```
TiFRONT(config-qos)# sh policy-map

  POLICY-MAP : test1
    CLASS-MAP : test1 precedence 1
        limit info :
          rate limit
              rate value: 500000
              burst value: 10000
```
- 결과
	- 400mbps / 손실률 37%
	- 500mbps / 손실률 53%
```
C:\Users\USER\Desktop\Dabin\iperf3>iperf3 -c 10.10.10.20 -u -b 400M -l 500 -t 10
Connecting to host 10.10.10.20, port 5201
[  5] local 10.10.10.10 port 54173 connected to 10.10.10.20 port 5201
[ ID] Interval           Transfer     Bitrate         Total Datagrams
[  5]   0.00-1.01   sec  48.0 MBytes   398 Mbits/sec  100592
[  5]   1.01-2.01   sec  47.1 MBytes   397 Mbits/sec  98765
[  5]   2.01-3.01   sec  48.3 MBytes   404 Mbits/sec  101237
[  5]   3.01-4.01   sec  47.1 MBytes   396 Mbits/sec  98751
[  5]   4.01-5.01   sec  48.3 MBytes   403 Mbits/sec  101242
[  5]   5.01-6.01   sec  47.1 MBytes   397 Mbits/sec  98675
[  5]   6.01-7.01   sec  48.3 MBytes   403 Mbits/sec  101336
[  5]   7.01-8.01   sec  47.9 MBytes   402 Mbits/sec  100476
[  5]   8.01-9.00   sec  46.6 MBytes   396 Mbits/sec  97702
[  5]   9.00-10.01  sec  48.7 MBytes   404 Mbits/sec  102084
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.01  sec   477 MBytes   400 Mbits/sec  0.000 ms  0/1000860 (0%)  sender
[  5]   0.00-10.01  sec   302 MBytes   253 Mbits/sec  0.037 ms  366727/1000860 (37%)  receiver

iperf Done.


```

- 값 변경
	- rate-limit: 800mbps
	- burst: 16mb
```
TiFRONT# sh policy-map

  POLICY-MAP : test1
    CLASS-MAP : test1 precedence 1
        limit info :
          rate limit
              rate value: 800000
              burst value: 16000
```
- 문제: 1G까지 전송 가능 (폴리싱 동작 안함)
```
C:\Users\USER\Desktop\Dabin\iperf3>iperf3 -c 10.10.10.20 -u -b 1000M -t 10
Connecting to host 10.10.10.20, port 5201
[  5] local 10.10.10.10 port 58399 connected to 10.10.10.20 port 5201
[ ID] Interval           Transfer     Bitrate         Total Datagrams
[  5]   0.00-1.00   sec   116 MBytes   966 Mbits/sec  30064
[  5]   1.00-2.00   sec   116 MBytes   977 Mbits/sec  30188
[  5]   2.00-3.01   sec   118 MBytes   979 Mbits/sec  30661
[  5]   3.01-4.00   sec   115 MBytes   976 Mbits/sec  29898
[  5]   4.00-5.01   sec   118 MBytes   977 Mbits/sec  30591
[  5]   5.01-6.01   sec   115 MBytes   972 Mbits/sec  29979
[  5]   6.01-7.00   sec   116 MBytes   977 Mbits/sec  30152
[  5]   7.00-8.01   sec   117 MBytes   976 Mbits/sec  30356
[  5]   8.01-9.01   sec   117 MBytes   977 Mbits/sec  30346
[  5]   9.01-10.00  sec   114 MBytes   966 Mbits/sec  29720
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.00  sec  1.13 GBytes   974 Mbits/sec  0.000 ms  0/301955 (0%)  sender
[  5]   0.00-10.01  sec  0.00 Bytes  0.00 bits/sec  0.000 ms  0/0 (0%)  receiver

iperf Done.


인터페이스 드롭 없음.
```
### 대역폭 제한 (shaping)
- 아래와 같이 설정
```
TiFRONT(config-qos)# service-queue output ge4 rate-limit 800000 128000

TiFRONT(config-qos)# show service-queue output ge4
 Service Queue Egress Setting...
   Interface: ge4
    SCHEDULE MODE: STRICT
    CoSq Rate Limit:
    -----------------------------------------
     CoS Q | min-rate(kbps)| max-rate(kbps)
    -------+---------------+-----------------
        0  |    no-limit   |    no-limit
        1  |    no-limit   |    no-limit
        2  |    no-limit   |    no-limit
        3  |    no-limit   |    no-limit
        4  |    no-limit   |    no-limit
        5  |    no-limit   |    no-limit
        6  |    no-limit   |    no-limit
        7  |    no-limit   |    no-limit
    -----------------------------------------
    Egress Rate Limit:
       Min-Rate(kbps): 800000, Max-Rate(kbps): 128000
```
- 결과
```
C:\Users\USER\Desktop\Dabin\iperf3>iperf3 -c 10.10.10.20 -u -b 2000M -t 10
Connecting to host 10.10.10.20, port 5201
[  5] local 10.10.10.10 port 51947 connected to 10.10.10.20 port 5201
[ ID] Interval           Transfer     Bitrate         Total Datagrams
[  5]   0.00-1.01   sec   116 MBytes   966 Mbits/sec  30113
[  5]   1.01-2.00   sec   116 MBytes   977 Mbits/sec  30115
[  5]   2.00-3.01   sec   117 MBytes   975 Mbits/sec  30531
[  5]   3.01-4.01   sec   116 MBytes   975 Mbits/sec  30119
[  5]   4.01-5.01   sec   117 MBytes   979 Mbits/sec  30351
[  5]   5.01-6.00   sec   116 MBytes   976 Mbits/sec  30074
[  5]   6.00-7.01   sec   118 MBytes   977 Mbits/sec  30544
[  5]   7.01-8.01   sec   116 MBytes   977 Mbits/sec  30116
[  5]   8.01-9.00   sec   116 MBytes   978 Mbits/sec  30177
[  5]   9.00-10.02  sec   118 MBytes   974 Mbits/sec  30552
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.02  sec  1.14 GBytes   975 Mbits/sec  0.000 ms  0/302692 (0%)  sender
[  5]   0.00-10.02  sec  0.00 Bytes  0.00 bits/sec  0.000 ms  0/0 (0%)  receiver

iperf Done.
```