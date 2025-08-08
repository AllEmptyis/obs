# ACL (Access List)
# 네트워크 접근 제어
- 특정 포트나 vlan에 대하여 접근/허용할 네트워크 대역을 설정
	- 사용자 트래픽 제어 용도
## TiFRONT ACL 설정 방법
- TiFRONT를 경유하는 패킷들의 **SIP, DIP, SPORT, DPORT, 프로토콜**을 검사하여 패킷을 필터링 하는 기능
- 보안 및 불필요한 트래픽 차단
- 동작 방식
	- ACL이 설정 된 포트 or vlan으로 패킷 수신 시
		- 설정 된 access list와 비교 후 일치 조건이 있다면 permit / deny
		- acl 조건은 하향식으로 적용 (순서 중요)
	- Access group
		- 여러 개의 access list 사용 시 하나의 그룹으로 묶어서 관리
### ACL 설정 방법
- ID(숫자,문자열) 각각 1000개의 ACL 정의 가능
	- 하나의 Access list에는 최대 50개의 규칙 설정 가능
- ACL 생성
	- access-list 
	- `<1-1000> | <word>` : ACL id 지정 (숫자 or 문자)
		- 숫자 범위: 1-1000, 문자:1-10자
	- `<1-1000>` : 규칙 우선순위
		- 동일한 우선순위 규칙이 있는 경우 새로 추가 된 규칙이 해당 우선순위로 적용
		- 우선순위 입력x -> 가장 낮은 우선순위 적용
	- `{deny | permit}` : 정책 지정
	- `<0-255> | any | tcp | udp` : 프로토콜 지정
	- `{<A.B.C.D/M> | any` : SIP 
	- `{<A.B.C.D/M> | any` : DIP
	- `{any | eq <1-65535> | range <1-65534> <2-65535>}`: SPORT, DPORT
	- log `<0-7>` : 기록할 로그 이벤트 레벨 설정
		- 0: Emergency, 1: Alert  2:Critical, 3:Error, 4: Warning 5:Notic 6:Information 7:Debug
		- 기본 값 : 3
	- interval `<1-600>` : 로그 기록 주기
- ACL 삭제
	- no access-list `id` : ACL 삭제
		- 인터페이스 or  access group에 적용되어 있는 경우 삭제 불가
- ACL 규칙 삭제
	- no access-list `규칙`
- ACL 적용 할 인터페이스 지정
	- access-list `<id>` interface `<ifname>`
		- 모든 패킷 차단 규칙이 마지막 규칙으로 추가 됨 (arp 제외)
- 참고
	- 한 인터페이스에는 하나의 ACL or Access group만 지정 가능
	- 포트/vlan 중 포트에 설정 된 ACL이 우선 적용
- 조회
	- `show access list interface`
### Access Group
- `access- group <word> access-list <id>`
- `access-group <word> interface <int>`
### 시간/날짜 기반 ACL
- `access-list <id> <time 0-24>`
- `access-list <id> <date> to <date>`
	- YYYY/MM/DD
# 시스템 접근 제어
- 장비 자체에 대한 접근 제어 (관리 접근 제한 용도)
	- 스위치 ip로 오는 패킷만 적용 (ex.스위치에 대한 ssh, icmp 등)
	- 우선순위는 시스템 접근 제어가 높음
		- ex. 네트워크 접근제어에서 all deny 설정 후 시스템 접근제어에서 허용 설정하면 적용.
- 특정 프로토콜에 대하여 접근/허용 할 네트워크 대역 설정
	- any, tcp, udp, icmp
- 호스트의 비밀번호가 유출되어 다른  사용자가 임의로 접근하는 것을 차단
	- 허용할 ip/네트워크 대역으로 지정
## 시스템 접근 제어 설정
- `system-access [deny/permit] [any/icmp/tcp/udp] [SIP] [DIP] [sport] [dport]`
- `show system-access`
# ACL 실습
## CLI mode
### 네트워크 접근 제어
- access list 설정
	- **모든 패킷에 대한 차단 규칙은 자동으로 생성**되기 때문에 넣을 필요 없음
	- 162 가상머신, 130 스위치 gw ip, 131 host
		- 호스트에서 가상머신으로 ssh 접속과 icmp 핑만 허용
```
access-list acl3 1 permit tcp 192.168.212.131/24 192.168.212.162/24 any eq 22
//출발지 포트는 any로 설정

TiFRONT(config)# show access-list acl3
 =========================================================================
   Access List acl3
     Action Time : any
     Action Date : any
        1 Action            : Permit
          IP Protocol       : 6
          Src IPv4 address  : 192.168.212.131/24
          Dst IPv4 address  : 192.168.212.162/24
          Dst port          : 22

        2 Action            : Permit
          IP Protocol       : 1
          Src IPv4 address  : 192.168.212.131/24
          Dst IPv4 address  : 192.168.212.162/24
 =========================================================================

TiFRONT(config)# access-list acl3 interface ge3
TiFRONT(config)#
TiFRONT(config)#
TiFRONT(config)# show access-list interface
 ---------------------------------
  Interface  |  Access-list (in)
 ------------+--------------------
        ge1  |         None
        ge2  |         None
        ge3  |         acl3
```
- access group 설정 (여기선 사용X)
```
TiFRONT(config)# access-group acg access-list acl
TiFRONT(config)# access-group acg access-list acl3

TiFRONT(config)# show access-group
 ============================================
   Access Group acg
     Action Time : None
     Action Date : None
       Access-list acl
       Access-list acl3
 ============================================

TiFRONT(config)# show access-group interface
 ----------------------------------
  Interface  |  Access-group (in)
 ------------+---------------------
        ge1  |          None
        ge2  |          None
        ge3  |           acg

//설정 조회
```
- 결과
	- 성공
		- 포트 변경 시 ping, ssh, 8443포트로 서버 접속 모두 가능
		- ge3번에 연결 시 ssh, ping만 가능 / 8443 포트 reset
### 시스템 접근 제어
- 호스트의 ip에 대해 설정 해도 적용 되지 않고 스위치 ip로의 접근만 차단 됨
```
TiFRONT(config)# system-access permit tcp 192.168.212.131/24 192.168.212.162/24 any eq 22
TiFRONT(config)# system-access deny tcp any 192.168.212.162/24 any any
TiFRONT(config)#
TiFRONT(config)#
TiFRONT(config)# show system-access
  System Access List
 -------------------------------------------------------
   #1
     Action            : Permit
     IP Protocol       : TCP
     Src IP address    : 192.168.212.131/24
     Dst IP address    : 192.168.212.162/24
     Src port          : ANY
     Dst port          :    22

   #2
     Action            : Deny
     IP Protocol       : TCP
     Src IP address    : ANY
     Dst IP address    : 192.168.212.162/24
     Src port          : ANY
     Dst port          : ANY
 -------------------------------------------------------
```
- 특정 호스트로부터 icmp 접근 차단
	- ip 대역은 서브넷 기준으로 처리, /24로 지정 시 해당 대역이 차단 됨
```
TiFRONT(config)# system-access deny icmp 192.168.212.131/32 192.168.212.130/24
TiFRONT(config)# show system-access
  System Access List
 -------------------------------------------------------
   #1
     Action            : Deny
     IP Protocol       : ICMP
     Src IP address    : 192.168.212.131/32
     Dst IP address    : 192.168.212.130/24
 -------------------------------------------------------
```
- 우선순위 비교
	- ping 안 됨 - 시스템 접근 제어 우세
```
TiFRONT(config)# show access-list permit
 =========================================================================
   Access List permit
     Action Time : any
     Action Date : any
        1 Action            : Permit
          IP Protocol       : 1
          Src IPv4 address  : 192.168.212.131/32
          Dst IPv4 address  : 192.168.212.130/24
 =========================================================================

TiFRONT(config)# show system-access
  System Access List
 -------------------------------------------------------
   #1
     Action            : Deny
     IP Protocol       : ICMP
     Src IP address    : 192.168.212.131/32
     Dst IP address    : 192.168.212.130/24
 -------------------------------------------------------
```
### 동작 원리
- 장비로 오는 패킷의 경우 시스템 접근제어 적용
	- 컨트롤 플레인에서 필터링
- 경유하는 패킷이면 네트워크 접근 제어 적용
	- 데이터 플레인에서 포워딩 결정
- 즉 장비로 오는 경우 시스템 접근 제어로 적용 된다
#### ticontroller
- 시스템 접근 제어는 커널의 방화벽에서 설정 됨 (iptables)
- 네트워크 접근 제어는 FP rule에 적용 됨. (스위칭 칩)
	- 네트워크 접근제어가 시스템접근제어와 겹치게 설정되어 있을 땐 시스템 접근 제어 동작 안 됨
		- ex) 네트워크 접근제어에서 all deny 했는데 시스템 접근제어는 all allow 한 경우
		- 시스템은 방화벽에서 막고, 네트워크 접근제어는 스위칭 칩에서 막아서 그런 거 같다. /네트워크에서 허용하고 시스템에서 deny하는 건 동작 잘 됨
```
bash-4.3# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all  --  TiFRONT              TiFRONT
ACCEPT     all  --  192.168.212.162      anywhere
ACCEPT     all  --  anywhere             192.168.212.162
ACCEPT     all  --  one.one.one.one      anywhere
ACCEPT     all  --  anywhere             one.one.one.one
ACCEPT     all  --  TiFRONT              anywhere
ACCEPT     all  --  anywhere             TiFRONT
ACCEPT     all  --  anywhere             anywhere
```

## Cloud mode
- 허용 주소
	- 컨트롤러 주소 외 기본적으로 허용 할 주소 설정 (최대 10개)
		- 컨트롤러 ip는 기본적으로 허용 됨
- 스위치 system-access 정책 확인
	- 컨트롤러 ip 트래픽 자동 정책 추가, 시스템 접근 제어에 규칙 하나라도 있으면 생성 됨.
	- 허용 주소에 설정한 ip에 대한 트래픽 허용 규칙도 자동 추가
```
TiFRONT(config)% show system-access
  System Access List
 -------------------------------------------------------
   #1
     Action            : Permit
     IP Protocol       : ANY
     Src IP address    : 192.168.212.162/32
     Dst IP address    : ANY

   #2
     Action            : Permit
     IP Protocol       : ANY
     Src IP address    : ANY
     Dst IP address    : 192.168.212.162/32

   #3
     Action            : Permit
     IP Protocol       : ANY
     Src IP address    : 127.0.0.1/32
     Dst IP address    : ANY

   #4
     Action            : Permit
     IP Protocol       : ANY
     Src IP address    : ANY
     Dst IP address    : 127.0.0.1/32

   #5
     Action            : Deny
     IP Protocol       : ICMP
     Src IP address    : 192.168.212.131/32
     Dst IP address    : ANY
 -------------------------------------------------------
TiFRONT(config)%
TiFRONT(config)%
TiFRONT(config)%
TiFRONT(config)%
TiFRONT(config)% show system-access
  System Access List
 -------------------------------------------------------
    None
 -------------------------------------------------------
```
- 네트워크 접근 제어 설정 시에는 스위치에 추가 되는 정책 없음
```
TiFRONT% show system-access
  System Access List
 -------------------------------------------------------
    None
 -------------------------------------------------------
TiFRONT%
TiFRONT% show access-list
 =========================================================================
   None
 =========================================================================
```