# 시스템 관리
## PLOS 업데이트
- os-update `<arguments>` `[mgmt]` 
	- os-update ftp://test:test123@192.168.212.225/PLOS-PASK-v2.2.7.4.0
- reboot
-----------------
# 기본 네트워크 설정
## IP 주소/라우팅 설정
- interface `<NAME>` ip `<add>`
	- no interface `<NAME>` ip `<ip>`
- route default-gateway `<Gateway>` 
- route network `<Dest>`  {gateway `<gateway>` interface `<int>`}: 고정 경로 추가 
- local ip

설정 확인
- show route
- show interface `<name>`
## 인터페이스 설정
- interface `<name>` status `up/down`
### mgmt 설정
- management-forward
- custom `<id>` : custom 설정 모드
- dip `<dip>`
## VLAN 생성
- vlan `<NAME>` vid `<vid>`
- vlan `<NAME>` port `<NAME>` `tagged/untagged`
## port 설정
- 포트 모드 변경
	- port  `1` sfp-mode copper
-------------------
# L4 부하 분산 설정
## Port-Boundary 설정
사용할 포트에 대해 포트 바운더리 설정이 필요
- port-boundary 1
- port `설정할 포트`
- promisc off
- apply
## real server 설정
- real `<id>`
- name `<name>`
- rip `<ip>`
- rpot `<rport>`
- apply
## SLB 설정
- slb `<name>`
	- ip-version `ipv4`
	- vip `<IP>` protocol `<protocol>` vport `<vport>`
	- nat-mode `l3dsr-iptunnel`
	- real `<ID>` 
	- real `<ID>` rport `<RPORT>`
	- health-check `<Health check id>`

- 설정 확인	
	- show slb: L4에 있는 slb 서비스 조회
	- show info slb `<name>` :  해당 slb 설정 정보 조회
	- currnet
	- apply

## H/C 설정
- slb 설정 중 health check 설정 방법
- 공통
	- health-check `<ID>`
	- type `설정`
	- interval `<1-60, 5>`
	- retry `<0-5, 0>`
	- port `<Port>` : HC 패킷 전송 할 포트 설정
	- sip `<SIP>` : HC 패킷 출발지
	- tip `<TIP>` 
	- apply
## ARP proxy
### proxy arp 설정
- arp proxy-arp enable
- show arp : 설정 확인
### arp filter 설정
#### 종류
- input: 다른 호스트가 pas k에게 mac 주소 요청 
- output: pas k가 mac 주소 요청
#### 명령어
- arp-filter
- input `<id>`
- action `accecpt / drop` : 정책 설정
- sip `<sip>`
- dip `<dip>`
- interface `<interface>` : 적용할 인터페이스
- apply
- show arp-filter
