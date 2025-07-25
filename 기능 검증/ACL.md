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
# 시스템 접근 제어
- 장비 자체에 대한 접근 제어 (관리 접근 제한 용도)
- 특정 프로토콜에 대하여 접근/허용 할 네트워크 대역 설정
	- any, tcp, udp, icmp
- 호스트의 비밀번호가 유출되어 다른  사용자가 임의로 접근하는 것을 차단
	- 허용할 ip/네트워크 대역으로 지정
## 시스템 접근 제어 설정
- `system-access [deny/permit] [any/icmp/tcp/udp] `
- `show system-access`
# ACL 실습
## CLI mode
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
          Src port          : 22
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
## 결과
- 성공
	- 포트 변경 시 ping, ssh, 8443포트로 서버 접속 모두 가능
	- ge3번에 연결 시 ssh, ping만 가능 / 8443 포트 reset