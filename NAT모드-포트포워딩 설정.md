## NAT 모드 포트포워딩
- NAT모드는 가상머신이 특정 대역에 격리 된 형태이기 때문에 외부에서 접근 불가
	- ip는 접근 불가하지만 특정 서비스 포트를 포트포워딩 하여 접근할 수 있도록 하는 방식
- 목표
	- 외부에서 vm으로 ssh 2222번 포트로 접속
### 설정 방식
1. iptables에서 설정
2. xml 파일 직접 수정
### 방화벽 / 포워딩 설정
- 방화벽에서 접근 할 포트 허용
	- firewall-cmd --permanent --add-port=2222/tcp
	- firewall-cmd --reload
- libvirt 포워딩 설정
	- iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 192.168.122.133:22
- DNAT (클라이언트가 vm으로 접근할 수 있도록)
	- -t nat: nat 테이블에서 작업
	- -A PREROUTING: 패킷이 들어오자마자 가장 먼저 걸리는 체인
	- -p tcp: tcp 프로토콜
	- --dport 2222: 목적지 포트 2222
	- -j DNAT: DNAT 수행
	- --to-destination `ip:port` : vm의 22번 포트로 포워딩
- SNAT (가상머신이 클라이언트로 응답하기 위한)
	- -A POSTROUTING: 패킷이 나가기 직전 체인
	- -d 192.168.122.x --dport 22: 목적지 ip / 포트가 22번인 경우\
	- -j MASQUERADE: SNAT 수행 (출발지 ip를 호스트 ip로)
## 동작 원리
- 외부 클라이언트가 호스트ip:2222로 요청
	- 내부에서 해당 요청에 대해 vmip:22로 포워딩 설정 필요 (DNAT)
- vm입장에서 패킷의 출발지는 외부ip:xxxx -> 모름
	- 출발지 ip를 호스트 ip로 SNAT 필요
```
[클라이언트] ---ssh---> [호스트IP:2222] ⇒ DNAT ⇒ [192.168.122.100:22 (VM)]

[VM] --응답--> [호스트 (SNAT)] --> [클라이언트]
```
## 실습
```
[root@localhost networks]# iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 192.168.122.133:22
[root@localhost networks]#  iptables -t nat -A POSTROUTING -p tcp -d 192.168.122.133 --dport 22 -j MASQUERADE

[root@localhost networks]# firewall-cmd --permanent --add-port=2222/tcp
success
[root@localhost networks]#
[root@localhost networks]# firewall-cmd --reload
success

[root@localhost networks]# iptables -L -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DNAT       tcp  --  anywhere             anywhere             tcp dpt:EtherNet/IP-1 to:192.168.122.133:22

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
LIBVIRT_PRT  all  --  anywhere             anywhere
MASQUERADE  tcp  --  anywhere             192.168.122.133      tcp dpt:ssh

Chain LIBVIRT_PRT (1 references)
target     prot opt source               destination
RETURN     all  --  192.168.122.0/24     base-address.mcast.net/24
RETURN     all  --  192.168.122.0/24     255.255.255.255
MASQUERADE  tcp  --  192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
MASQUERADE  udp  --  192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
MASQUERADE  all  --  192.168.122.0/24    !192.168.122.0/24

firwall-cmd --list-all
```
### 오류 (접속 안 됨)
- 호스트 ip 2222번 포트로 요청까지 들어오지만 virbr0에서는 캡쳐 안 됨
```
[root@localhost networks]# tcpdump -ni br0 port 2222
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on br0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:11:23.585748 IP 192.168.212.184.63047 > 192.168.193.100.EtherNet/IP-1: Flags [S], seq 3699090133, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
16:11:27.590177 IP 192.168.212.184.63047 > 192.168.193.100.EtherNet/IP-1: Flags [S], seq 3699090133, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
```
- 해결 방법
	- VIP 설정 등등 해보았으나 해결 안 됨
	- 브릿지 구성 삭제 후 NAT 모드에서만 실행하기
### 해결
- iptables 확인
	- `LIBVIRT_FWO` 체인에 패킷 들어오지 않음 -> 즉 VM에서 외부로 응답 못 함
```
[root@localhost ~]# iptables -t filter -L FORWARD -v
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
   10   520 LIBVIRT_FWX  all  --  any    any     anywhere             anywhere
   10   520 LIBVIRT_FWI  all  --  any    any     anywhere             anywhere
    0     0 LIBVIRT_FWO  all  --  any    any     anywhere             anywhere
```
- 기본 룰
	- cstate RELATED,ESTABLISHED -> 외부에서 vm으로 들어오는 요청은 이미 연결 된 요청에 한해서만 허용, 즉 신규 요청 거부
	- iptables -I FORWARD -p tcp -d 192.168.122.133 --dport 22 -j ACCEPT 설정 필요
		- DNAT 후  122.133 이 목적지인 경우 포워딩 허용
```
Chain LIBVIRT_FWI (1 references)
target     prot opt source               destination
ACCEPT     all  --  anywhere             192.168.122.0/24     ctstate RELATED,ESTABLISHED
REJECT     all  --  anywhere             anywhere             reject-with icmp-port-unreachable

Chain LIBVIRT_FWO (1 references)
target     prot opt source               destination
ACCEPT     all  --  192.168.122.0/24     anywhere
REJECT     all  --  anywhere             anywhere             reject-with icmp-port-unreachable
```
- 확인
	- virbr0, vnet8에서는 dnat 되어 전송 요청 들어옴
	- eno1에서는 nat 되기 전 호스트ip:2222로 요청 전송
```
[root@localhost ~]# tcpdump -ni virbr0 port 22
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on virbr0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:25:26.085681 IP 192.168.212.193.49317 > 192.168.122.133.ssh: Flags [S], seq 829648634, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
16:25:26.085969 IP 192.168.122.133.ssh > 192.168.212.193.49317: Flags [S.], seq 786019950, ack 829648635, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
16:25:26.086502 IP 192.168.212.193.49317 > 192.168.122.133.ssh: Flags [.], ack 1, win 255, length 0
16:25:26.089851 IP 192.168.212.193.49317 > 192.168.122.133.ssh: Flags [P.], seq 1:29, ack 1, win 255, length 28: SSH: SSH-2.0-PuTTY_Release_0.83
16:25:26.090064 IP 192.168.122.133.ssh > 192.168.212.193.49317: Flags [.], ack 29, win 502, length 0

[root@localhost ~]# tcpdump -ni vnet8 port 22
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on vnet8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:27:15.898417 IP 192.168.212.193.49327 > 192.168.122.133.ssh: Flags [S], seq 3941279771, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
16:27:15.898697 IP 192.168.122.133.ssh > 192.168.212.193.49327: Flags [S.], seq 1849847722, ack 3941279772, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
16:27:15.899296 IP 192.168.212.193.49327 > 192.168.122.133.ssh: Flags [.], ack 1, win 255, length 0

[root@localhost ~]# tcpdump -ni eno1 port 2222
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:27:37.768863 IP 192.168.212.193.49330 > 192.168.193.100.EtherNet/IP-1: Flags [S], seq 2676337853, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
16:27:37.769182 IP 192.168.193.100.EtherNet/IP-1 > 192.168.212.193.49330: Flags [S.], seq 980654235, ack 2676337854, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
16:27:37.769732 IP 192.168.212.193.49330 > 192.168.193.100.EtherNet/IP-1: Flags [.], ack 1, win 255, length 0
16:27:37.774201 IP 192.168.212.193.49330 > 192.168.193.100.EtherNet/IP-1: Flags [P.], seq 1:29, ack 1, win 255, length 28
16:27:37.774426 IP 192.168.193.100.EtherNet/IP-1 > 192.168.212.193.49330: Flags [.], ack 29, win 502, length 0
```
