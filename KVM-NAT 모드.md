## 사용 용도
- 리눅스를 VPN 서버로 구성 시
- 컨테이너 등 가상화 환경
## 실습
### 통신 확인
- 내부통신 (vm->host ping)
	- virbr0, vnet에만 패킷 찍힘
- 외부통신 (vm->8.8.8.8 ping)
	- 외부로 나갈 땐 호스트 커널의 iptables/NAT가 SNAT 해서 전송
	- vm >virbr0 >호스트 커널에서 SIP NAT >eno1>외부
```
[root@localhost networks]# tcpdump -ni br0 -v icmp  --->외부와 통신할 땐 호스트 ip로 통신 / 외부로 나갈 땐 호스트 커널의 iptables/NAT가 SNAT 해서 전송
dropped privs to tcpdump
tcpdump: listening on br0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:08:11.411171 IP (tos 0x0, ttl 63, id 63821, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.193.100 > 8.8.8.8: ICMP echo request, id 3, seq 5, length 64
14:08:11.443435 IP (tos 0x0, ttl 106, id 0, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.193.100: ICMP echo reply, id 3, seq 5, length 64
14:08:12.412867 IP (tos 0x0, ttl 63, id 64441, offset 0, flags [DF], proto ICMP (1), length 84)

[root@localhost networks]# tcpdump -ni virbr0  --->122.x 인터페이스 / MAT 전
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on virbr0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:08:21.426932 IP 192.168.122.133 > 8.8.8.8: ICMP echo request, id 3, seq 15, length 64
14:08:21.458960 IP 8.8.8.8 > 192.168.122.133: ICMP echo reply, id 3, seq 15, length 64

[root@localhost networks]# tcpdump -ni eno1 -v icmp
dropped privs to tcpdump
tcpdump: listening on eno1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:16:16.472487 IP (tos 0x0, ttl 63, id 26317, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.193.100 > 8.8.8.8: ICMP echo request, id 4, seq 10, length 64
14:16:16.508033 IP (tos 0x0, ttl 105, id 0, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.193.100: ICMP echo reply, id 4, seq 10, length 64
14:16:17.474489 IP (tos 0x0, ttl 63, id 26960, offset 0, flags [DF], proto ICMP (1), length 84)
```