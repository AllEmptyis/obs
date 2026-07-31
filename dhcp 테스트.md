세팅
vlan10:ge7,ge11

- 공유기 원리
	- 내부적으로 dhcp 서버가 있어서 자신이 할당한 호스트가 외부로 통신을 보내면 wan ip로 nat 해서 보냄
	- 이 때 wan ip는 공유기에 연결된 wan (실제 dhcp 서버로부터 할당 받은 것)
```
bash-4.3# tcpdump -nei vlan10 icmp

09:33:39.973885 00:06:c4:55:05:4c > 00:26:66:f6:83:f1, ethertype IPv4 (0x0800), length 74: 10.10.10.1 > 10.10.10.6: ICMP echo reply, id 1, seq 104, length 40
09:33:40.994447 00:26:66:f6:83:f1 > 00:06:c4:55:05:4c, ethertype IPv4 (0x0800), length 78: 10.10.10.6 > 10.10.10.1: ICMP echo request, id 1, seq 105, length 40
```


1. 정상적으로 dhcp 서버로부터 받아오는 경우
2. 공유기 대역 받아오는 경우
3. 공유기 loop 발생 시켰을 경우