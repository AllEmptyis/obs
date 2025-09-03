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
- 정책 - 클래스 별로 적용 될 정책 정의
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
	- QoS 처리 과정
		- 분류: dscp, cos등을 보고 어떤 큐에 넣을지 결정
		- 스케줄링: 각 큐에 있는 패킷을 어떤 순서/비율로 꺼낼지 결정
			- 즉 패킷이 우선순위 높은 큐로 가도 스케줄링을 rr로 하면 qos 효과가 별로 없음
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
```
## 설정 -CLI
- class map
	- qos
	- class-map `<map name>`
	- match `<분류 기준 정의>`
- policy map (confirm-action / rate-limit 두 종류)
	- qos
	- policy-map `<policy name>`
	- class `<적용할 클래스명> <우선순위>`
	- confirm-action `qos action`
		- **패킷 차단/허용 , dscp 값 삽입, 우선순위 설정 등**
	- rate-limit `<클래스의 트래픽에 보장해줄 최대 대역폭> <클래스의 트래픽이 사용할 수 있는 최대 버스트>`
		- 버스트: 순간적으로 몰리는 트래픽량 허용 범위
		- 최대 대역폭 설정보다 큰 트래픽이 들어오게 되면 허용된 버스트 만큼 잠시 허용
- service policy (적용)
	- qos
	- service-policy `<정책 이름>`
	- 한 정책만 적용 가능
	- 정책이 변경된 경우 servie policy로 재적용 필요
- 큐 스케줄링 설정
	- **service-queue output `<ifname>` schedule mode `{drr/rr/spq/wrr}`**
		- 스케줄 모드
			- sqp: 기본 / 우선순위 높은 패킷 부터 처리
			- rr: 각 큐를 순차적으로 처리
			- drr: 지정된 weight에 따라 프레임 사이즈 별로 라운드 로빈
			- wrr: 지정된 weight에 따라 프레임 별로 라운드 로빈
	- service-queue input `<ifname>` cos-map `{defualt/Cos우선순위/큐번호}`
		- CoS 필드의 우선순위에 따라 전송 큐 처리하도록 설정
```
RTK: servie-queue 명령어 input만 가능
BCM: input/output 모두 가능
- 즉 bcm에서는 qos 설정 ingress/egress 모두 가능한 것으로 보여짐. (확인 필요)
```
- 포트 별 송/수신 대역폭 제한 설정
	- service-queue output `<ifname>` rate-limit `{<최대대역폭 최대버스트>/none}`
- 큐 별 속도 제한
	- service-queue output `<ifname>` cos-rate-limit `<큐번호> {<최소 대역폭 최대 대역폭>/none}`
- 조회
	- show class-map / policy-map / service-policy
	- show service-queue `input/output <ifname>`
	- show qos install
## 설정 - Ticontroller
- ACL 설정에서 네트워크 접근 제어 설정할 때 qos 설정 가능
	- dscp, cos, rate-limit 설정 가능
		- rate-limit만 설정한 경우 설정한 값에 맞게 트래픽 인가
		- dscp,cos와 함께 rate-limit 설정한 경우: dscp 값이 매칭 되는 트래픽에 대해 rate-limit 동작 ->왜 cos값은 안보는지?
### 컨트롤러 동작 방식
- CLI mode에서는 qos, acl 각각 다른 fp group으로 처리됨
- cloud mode에서는 qos, acl fp group이 합쳐지면서 qos 기능 감소
	- https://redmine.piolink.com/issues/56434
- QoS/ACL fp group 통합
	- 클라우드 모드로 사용 중 cli로 qos 설정 넣으면 네트워크접근제어(acl)과 중복되어 오류 발생 가능
	- https://redmine.piolink.com/issues/105935
- json형식으로 스위치에 전달
	- QoS/ACL에 대한 매칭 카운트
	- count get/set/reset
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
- class별 rate limit 정책 설정
	- 최대 대역폭: 200mbps / 버스트: 16m
```
TiFRONT(config-qos)# show qos install
  SERVICE POLICY : test1

  CLASS-MAP : test1 precedence 6
    match :
        any enable
    action :
        limit info :
          rate limit
              rate value: 200000
              burst value: 16000
```
- 대역폭 제한 없는 경우
	- 비트레이트: 약 300mbps
```
C:\Users\parkd\Desktop\case>iperf3 -c 10.10.10.10 -u -b 1000M -l 500 -t 10
Connecting to host 10.10.10.10, port 5201
[  5] local 10.10.10.20 port 50468 connected to 10.10.10.10 port 5201
[ ID] Interval           Transfer     Bitrate         Total Datagrams
[  5]   0.00-1.01   sec  38.4 MBytes   320 Mbits/sec  80495
[  5]   1.01-2.01   sec  38.3 MBytes   320 Mbits/sec  80229
[  5]   2.01-3.01   sec  38.2 MBytes   321 Mbits/sec  80149
[  5]   3.01-4.01   sec  38.5 MBytes   321 Mbits/sec  80767
[  5]   4.01-5.01   sec  38.2 MBytes   321 Mbits/sec  80121
[  5]   5.01-6.00   sec  38.1 MBytes   322 Mbits/sec  79798
[  5]   6.00-7.01   sec  38.4 MBytes   321 Mbits/sec  80580
[  5]   7.01-8.01   sec  38.7 MBytes   322 Mbits/sec  81078
[  5]   8.01-9.01   sec  38.3 MBytes   323 Mbits/sec  80286
[  5]   9.01-10.02  sec  38.7 MBytes   322 Mbits/sec  81073
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.02  sec   384 MBytes   321 Mbits/sec  0.000 ms  0/804576 (0%)  sender
[  5]   0.00-10.02  sec   377 MBytes   316 Mbits/sec  0.023 ms  13482/804576 (1.7%)  receiver

iperf Done.
```
- 대역폭 제한 설정
	- 서버
```
C:\Users\parkd\Desktop\case>iperf3 -c 10.10.10.10 -u -b 1000M -l 1400 -t 10
Connecting to host 10.10.10.10, port 5201
[  5] local 10.10.10.20 port 60752 connected to 10.10.10.10 port 5201
[ ID] Interval           Transfer     Bitrate         Total Datagrams
[  5]   0.00-1.00   sec  41.7 MBytes   348 Mbits/sec  31214
[  5]   1.00-2.01   sec  41.5 MBytes   345 Mbits/sec  31110
[  5]   2.01-3.01   sec  41.0 MBytes   346 Mbits/sec  30731
[  5]   3.01-4.00   sec  40.9 MBytes   345 Mbits/sec  30669
[  5]   4.00-5.01   sec  41.7 MBytes   346 Mbits/sec  31201
[  5]   5.01-6.01   sec  41.3 MBytes   346 Mbits/sec  30919
[  5]   6.01-7.01   sec  40.7 MBytes   344 Mbits/sec  30491
[  5]   7.01-8.00   sec  41.2 MBytes   346 Mbits/sec  30894
[  5]   8.00-9.01   sec  41.3 MBytes   346 Mbits/sec  30957
[  5]   9.01-10.01  sec  41.5 MBytes   348 Mbits/sec  31052
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.01  sec   413 MBytes   346 Mbits/sec  0.000 ms  0/309238 (0%)  sender
[  5]   0.00-10.21  sec   233 MBytes   191 Mbits/sec  0.076 ms  134754/309237 (44%)  receiver

iperf Done.
```
- 클라이언트
```
C:\Users\USER\Desktop\Dabin\iperf3>iperf3 -s
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 10.10.10.20, port 51360
[  5] local 10.10.10.10 port 5201 connected to 10.10.10.20 port 60752
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-1.00   sec  25.1 MBytes   210 Mbits/sec  0.072 ms  12312/31084 (40%)
[  5]   1.00-2.01   sec  23.3 MBytes   194 Mbits/sec  0.096 ms  13677/31110 (44%)
[  5]   2.01-3.00   sec  22.9 MBytes   194 Mbits/sec  0.065 ms  13461/30593 (44%)
[  5]   3.00-4.01   sec  23.3 MBytes   194 Mbits/sec  0.112 ms  13662/31094 (44%)
[  5]   4.01-5.00   sec  22.9 MBytes   194 Mbits/sec  0.080 ms  13506/30648 (44%)
[  5]   5.00-6.01   sec  23.2 MBytes   194 Mbits/sec  0.067 ms  13714/31110 (44%)
[  5]   6.01-7.00   sec  22.9 MBytes   194 Mbits/sec  0.137 ms  13319/30495 (44%)
[  5]   7.00-8.01   sec  23.3 MBytes   194 Mbits/sec  0.064 ms  13703/31141 (44%)
[  5]   8.01-9.01   sec  23.0 MBytes   194 Mbits/sec  0.112 ms  13569/30821 (44%)
[  5]   9.01-10.01  sec  23.1 MBytes   193 Mbits/sec  0.076 ms  13831/31141 (44%)
[  5]  10.01-10.21  sec  0.00 Bytes  0.00 bits/sec  0.076 ms  0/0 (0%)
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.21  sec   233 MBytes   191 Mbits/sec  0.076 ms  134754/309237 (44%)  receiver
```
- 결과 요약
	- bitrate 200mbps로 제한 됨
	- sender는 똑같이 300mbps로 전송하지만, 수신측에서 레이트리밋 걸림
## 큐 스케줄링
- 포트 속도 100mbps로 제한 후 큐 스케줄링 테스트 (우선순위 높은 것만)

# FP group 확인 (BCM)
## qos 설정 없을 때 / CLI
```
TiFRONT# show sw-fabric-resource filter
 -----------------------------------------------------------------
                SWITCH FABRIC RESOURCE FILTER
 -----------------------------------------------------------------
                                         ENTRY  (2048) / USE( 61%)
                                         COUNTER( 896) / USE( 85%)
                                         METER  ( 768) / USE(  0%)
 -----------------------------------------------------------------
   CATEGORY |        NAME      |    USED   |  AVAILABLE  |  USE(%)
 -----------+------------------+-----------+-------------+--------
      ENTRY |       SECIPv4(0) |    512(D) |         768 |    25%
    COUNTER |       SECIPv4(0) |    249(D) |         134 |    27%
      METER |       SECIPv4(0) |      0(D) |         768 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |           SWT(1) |    256(D) |         768 |    12%
    COUNTER |           SWT(1) |      1(D) |         134 |     0%
      METER |           SWT(1) |      0(D) |         640 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |      TiNDM-RX(4) |    256(S) |         768 |    12%
    COUNTER |      TiNDM-RX(4) |    256(S) |         134 |    28%
      METER |      TiNDM-RX(4) |      0(S) |        1024 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |      TiNDM-TX(5) |    256(S) |         768 |    12%
    COUNTER |      TiNDM-TX(5) |    256(S) |         134 |    28%
      METER |      TiNDM-TX(5) |      0(S) |        1024 |     0%
 -----------+------------------+-----------+-------------+--------
  CATEGORY  : ENTRY / COUNTER / METER
  NAME      : GROUP NAME(Group ID)
  USED      : USE COUNT(Q:Quad / T:Triple+128 / D:Double / S:Single)
  AVAILABLE : AVAILABLE COUNT
  USE       : USE PERCENTAGE
```
## qos 설정 적용 / CLI
- QOS-ACLIPv4(2) : 해당 fp group 사용
- entry: 룰을 몇 개 설정할 수 있는지
- meter: 트래픽 제한용 필드
```
TiFRONT(config-qos)# sh qos install
  SERVICE POLICY : test

  CLASS-MAP : test precedence 6
    match :
        any enable
    action :
        commit info :
          permit

TiFRONT# sh sw-fabric-resource filter
 -----------------------------------------------------------------
                SWITCH FABRIC RESOURCE FILTER
 -----------------------------------------------------------------
                                         ENTRY  (2048) / USE( 73%) -->증가
                                         COUNTER(1024) / USE( 86%)
                                         METER  ( 640) / USE(  0%)
 -----------------------------------------------------------------
   CATEGORY |        NAME      |    USED   |  AVAILABLE  |  USE(%)
 -----------+------------------+-----------+-------------+--------
      ENTRY |       SECIPv4(0) |    512(D) |         512 |    25%
    COUNTER |       SECIPv4(0) |    249(D) |         134 |    24%
      METER |       SECIPv4(0) |      0(D) |         640 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |           SWT(1) |    256(D) |         512 |    12%
    COUNTER |           SWT(1) |      1(D) |         134 |     0%
      METER |           SWT(1) |      0(D) |         512 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |   QOS-ACLIPv4(2) |    256(D) |         512 |    12%
    COUNTER |   QOS-ACLIPv4(2) |    128(D) |         134 |    12%
      METER |   QOS-ACLIPv4(2) |      0(D) |         512 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |      TiNDM-RX(4) |    256(S) |         512 |    12%
    COUNTER |      TiNDM-RX(4) |    256(S) |         134 |    25%
      METER |      TiNDM-RX(4) |      0(S) |         768 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |      TiNDM-TX(5) |    256(S) |         512 |    12%
    COUNTER |      TiNDM-TX(5) |    256(S) |         134 |    25%
      METER |      TiNDM-TX(5) |      0(S) |         768 |     0%
 -----------+------------------+-----------+-------------+--------
```
- ratelimit 설정한 경우 meter 로 카운트 됨
```
TiFRONT# sh sw-fabric-resource filter
 -----------------------------------------------------------------
                SWITCH FABRIC RESOURCE FILTER
 -----------------------------------------------------------------
                                         ENTRY  (2048) / USE( 73%)
                                         COUNTER(1024) / USE( 86%)
                                         METER  ( 640) / USE(  0%)
 -----------------------------------------------------------------
   CATEGORY |        NAME      |    USED   |  AVAILABLE  |  USE(%)
 -----------+------------------+-----------+-------------+--------
      ENTRY |       SECIPv4(0) |    512(D) |         512 |    25%
    COUNTER |       SECIPv4(0) |    249(D) |         134 |    24%
      METER |       SECIPv4(0) |      0(D) |         640 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |           SWT(1) |    256(D) |         512 |    12%
    COUNTER |           SWT(1) |      1(D) |         134 |     0%
      METER |           SWT(1) |      0(D) |         512 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |   QOS-ACLIPv4(2) |    256(D) |         512 |    12%
    COUNTER |   QOS-ACLIPv4(2) |    128(D) |         134 |    12%
      METER |   QOS-ACLIPv4(2) |      1(D) |         511 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |      TiNDM-RX(4) |    256(S) |         512 |    12%
    COUNTER |      TiNDM-RX(4) |    256(S) |         134 |    25%
      METER |      TiNDM-RX(4) |      0(S) |         768 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |      TiNDM-TX(5) |    256(S) |         512 |    12%
    COUNTER |      TiNDM-TX(5) |    256(S) |         134 |    25%
      METER |      TiNDM-TX(5) |      0(S) |         768 |     0%
 -----------+------------------+-----------+-------------+--------
```
- 의문
	- acl과 같은 fp group에 적용되는것인지? acl 설정 했을 때는 fp group에서 use가 안올라감
## qos 설정 적용 / Cloud
- fp 그룹명 ACLIPv4(2)
```
TiFRONT# sh sw-fabric-resource filter
 -----------------------------------------------------------------
                SWITCH FABRIC RESOURCE FILTER
 -----------------------------------------------------------------
                                         ENTRY  (2048) / USE( 67%)
                                         COUNTER(1024) / USE( 87%)
                                         METER  ( 640) / USE(  0%)
 -----------------------------------------------------------------
   CATEGORY |        NAME      |    USED   |  AVAILABLE  |  USE(%)
 -----------+------------------+-----------+-------------+--------
      ENTRY |       SECIPv4(0) |    512(D) |         512 |    25%
    COUNTER |       SECIPv4(0) |    256(D) |         127 |    25%
      METER |       SECIPv4(0) |      0(D) |         640 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |           SWT(1) |    256(D) |         512 |    12%
    COUNTER |           SWT(1) |      1(D) |         127 |     0%
      METER |           SWT(1) |      0(D) |         512 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |       ACLIPv4(2) |    128(S) |         640 |     6%
    COUNTER |       ACLIPv4(2) |    128(S) |         127 |    12%
      METER |       ACLIPv4(2) |      0(S) |         768 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |      TiNDM-RX(4) |    256(S) |         640 |    12%
    COUNTER |      TiNDM-RX(4) |    256(S) |         127 |    25%
      METER |      TiNDM-RX(4) |      0(S) |         896 |     0%
 -----------+------------------+-----------+-------------+--------
      ENTRY |      TiNDM-TX(5) |    256(S) |         640 |    12%
    COUNTER |      TiNDM-TX(5) |    256(S) |         127 |    25%
      METER |      TiNDM-TX(5) |      0(S) |         896 |     0%
 -----------+------------------+-----------+-------------+--------
  CATEGORY  : ENTRY / COUNTER / METER
  NAME      : GROUP NAME(Group ID)
  USED      : USE COUNT(Q:Quad / T:Triple+128 / D:Double / S:Single)
  AVAILABLE : AVAILABLE COUNT
  USE       : USE PERCENTAGE
```

-----
https://atthis.tistory.com/3