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
- L4는 서버에게 IP 터널링으로 패킷을 캡슐화하여 전달하고, 응답할 땐 서버가 직접 원래 클라이언트의 ip로 전송
#### 대표 기술
- IP in IP: ip 캡슐화
- VXLAN

------------
# 신규 기능
## IP 터널에 대한 TCP/UDP healthcheck 기능 개발
##### 기능 개발 히스토리
- DSR 구성(ip터널/dscp)에서 SLB VIP로 TCP/UDP health check 기능 개발 요청
##### 기존 기능
- icmp 통한 H/C
- 



------------
https://redmine.piolink.com/issues/165787