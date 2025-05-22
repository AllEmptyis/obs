## IP Tunnel DSR란?
### DSR (Direct Server Return)
#### 개념
- 클라이언트는 L4를 거쳐 서버로 전달
- 서버는 L4를 거치지 않고 **직접 클라이언트에게 전달하는 방식**
#### 장점
- L4 트래픽 부담 감소
- 고성능, 대용량 트래픽 처리에 유리
### IP Tunnel
#### 개념
- IP 패킷을 다른 프로토콜로 캡슐화하여 통신하도록 하는 기술
- IP in IP
	- inner IP: 실제 목적지
	- outer IP: 캡슐화 되는 IP, 터널을 지날 때 사용 
#### 사용 이유
- 사설망으로 통신 (VPN, IPsec 등)
- 네트워크 경로를 숨기기 위해

## IP Tunnel 방식의 DSR
- DSR 구현 방식 중 하나, IP in IP encapsulation
### 구조
- 클라이언트는 L4로 요청
- **L4는 서버에게 IP 터널링으로 패킷을 캡슐화하여 전달하고, 응답할 땐 서버가 직접 원래 클라이언트의 ip로 전송**
- 패킷의 MAC주소만 바꾸는 방식 (IP는 캡슐화)
	- 캡슐화 된 ip에 대하여는 arp 응답하지 않도록 설정 필요
#### 대표 기술
- IP in IP: ip 캡슐화
- VXLAN
#### 동작 방식
1) 클라이언트는 vip:vport로 요청을 전송
	- 서버에서 vip에 대해 arp 응답하지 않도록 설정 필요 (클라이언트가 vip로 요청 시 서버가 응답 방지)
2) L4가 받아서 적절한 서버로 LB
3) 선택 된 서버로 패킷을 ipip해서 전달 (실제 outer ip로 라우팅 된다)
	- outer: src(slb tunnel ip) / dst(server tunnel ip, 보통 rip)
	- inner: src(client ip) / dst(slb vip)
	- **이 때, 서버의 [[Loopback 인터페이스]]에 vip가 바인딩 되어 있어야 디캡 후 서버가 받을 수 있다**
4) 서버가 패킷을 받은 후 디캡하여 클라이언트로 직접 패킷 전송
	- src: vip / dst: 클라이언트 ip (inner ip에 있던 정보로 알게 됨)
#### H/C 동작
- icmp 방식
	- 리얼서버에게 rip(vip)로 ping 하여 확인
	- outer: rip / inner: vip
	- 디캡했을 때 vip에 대해 응답하는지 확인 (vip로 응답)
- tcp 방식
	- 리얼서버의 rip:rport(vip:vport)로 syn 던져서 확인

-------------------
# 신규 기능 
## IP 터널에 대한 TCP/UDP healthcheck 기능 개발 (#165787)
##### 버전
**v2.2.7.4.0**
##### 기능 개발 히스토리
- DSR 구성(ip터널/dscp)에서 SLB VIP로 TCP/UDP health check 기능 개발 요청
	- 참고: [[PAS-K 서버 부하 분산 NAT 모드]]
##### 기존 기능
- icmp 통한 H/C
	- 캡슐화 할 ip(주로 slb의 vip)에 대해서 리얼서버가 잘 응답하는지 확인
	- 직접 real로 ping하면 응답하지 않기 때문에, capsulation해서 vip에 대해 잘 응답하는지 확인 
		- outer: 리얼서버의 rip
		- inner: 확인 할 실제 vip
##### 용어
- **Inner TIP**: 캡슐화 되는 ip의 목적지 ip (inner dip) / 보통 SLB의 vip
##### 동작 원리
- icmp와 동일하게 확인 할 vip를 캡슐화 하여 real ip로 syn을 보내어 H/C
- 만일 해당 포트에 대하여 Reset이 오면 포트가 닫혀있는 것

## 실습
### 실습 과정
- pas-k
	1) vlan 생성, ip, 헬스체크, 리얼서버 설정
	2) slb 구성
- real server
	1) 루프백 인터페이스에 slb vip 설정
	2) 리얼서버가 루프백 인터페이스에 설정한 vip에 대해 arp 응답하지 않도록 설정
	3) 리스닝 포트 설정
### 실습 구성
- pas-k
	```
	vlan 인터페이스 - default(vlan 1), test (vlan193)
	test: 192.168.193.181/24

	slb: 
	rip - 192.168.211.150 (server)
	vip - 192.168.193.50
	```
- L2
	```
	vlan 211 에 L4 연결 포트 (xg4)
	```
- real server
	- 192.168.211.150
- client
	- 192.168.212.225
- [[실습 과정]]

------------
https://redmine.piolink.com/issues/165787