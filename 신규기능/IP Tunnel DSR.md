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
2) L4가 리얼서버에서 vip로 캡슐화 하여 mac만 리얼서버의 mac으로 전달 
3) 리얼서버는 vip가 [[Loopback 인터페이스 | Loopback]] 인터페이스에 설정 되어 있어 해당 패킷을 받음
	- 이 때, vip에 대해 arp 응답 방지 설정을 하지 않으면 클라이언트가 vip로 응답할 때 리얼서버가 응답할 수 있다
4) 패킷을 디캡슐레이션 하여 리얼서버가 직접 클라이언트에게 **vip로 응답** 
	- **따라서 L3 구간인 경우 리얼서버의 mac, ip 모두 노출되지 않는다**
#### H/C 동작
- L4는 리얼서버의 mac을 헬스체크를 통해 알아 옴
- HC 패킷을 IPIP 하여 서버에게 전달, 서버의 응답을 통해 mac 주소를 알게 된다.
- icmp 방식
	- 리얼서버에게 vip로 ping 하여 확인
- tcp 방식
	- 리얼서버의 리스팅 포트(rport)로 syn 던져서 확인

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
- **Inner TIP**: 실제 목적지 ip를 캡슐화 할 ip / 보통 SLB의 vip
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
	vlan 인터페이스 - default(vlan 1), test (vlan10)
	default: 10.10.10.1
	test: 192.168.193.181/24

	slb: 
	rip - 10.10.10.10
	vip - 192.168.193.50
	```
- L2
	```
	vlan 211 에 L4 연결 포트 (xg4)
	10.10.10.0 대역에 대해서 라우팅 설정
	```
- real server
	- 10.10.10.10
- client
	- 192.168.212.225
- [[실습 과정]]

------------
https://redmine.piolink.com/issues/165787