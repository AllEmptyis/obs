---
aliases:
  - IP Tunnel DSR_H/C 실습
  - IP Tunnel DSR_H/C 실습
---
## 실습 단계
- [x] PLOS 업데이트
	```
	SBM_TEST(config)# os-update ftp://test:test123@192.168.212.225/PLOS-PASK-v2.2.7.4.0
	Firmware tool: Updating Firmware

	BEGIN: Downloading via FTP (2025-05-15 11:25:25,372)
	- [########################################] 100.0% (887,164,053 / 887,164,053)
	```
- [x] 라우팅 설정
- [x] SLB 생성
- [x] 서버 설정
	- [x] nginx 설치
- [x] SLB 확인
- [x] H/C 확인

## 구성(tcp)
### PAS-K
- 버전 확인
	```
	switch# show system
	
	================================================================================
	  SYSTEM
	 ------------------------------------------------------------------------------
	    System
	        Product Name     : PAS-K3200X
	        Serial Number    : R211K30000A03201
	        OS version       : v2.2.7.4.0
	        Mgmt IP Address  : 192.168.100.1/24
	        Mgmt MAC Address : 00:06:c4:90:03:92
	        SDRAM Size       : 16384MB
	        Storage Size     : 256.1GB
	        Log Storage Size : 212.9GB (used: 2.9G (2%))
	        System Uptime    : 2 minutes 27 seconds
	================================================================================
	```
- 라우팅 설정
	```
	PAS-K(config)# route network 192.168.212.0/24 gateway 192.168.193.1
	PAS-K(config)# show interface
	
	================================================================================
	INTERFACE Configuration
	 ------------------------------------------------------------------------------
	  Name    Status MAC Address       IPv4 Address       Broadcast       RPF     Description
	  test    up     00:06:c4:90:03:93 192.168.193.181/24 192.168.193.255 default
	  default up     00:06:c4:90:03:93                                    default
	  mgmt    up     00:06:c4:90:03:92 192.168.100.1/24   192.168.100.255 default
	================================================================================
	
	PAS-K(config)# show route
	
	================================================================================
	  ROUTE
	 ------------------------------------------------------------------------------
	    Default-Gateway      : 192.168.193.1
	
	    Network
	       Destination      Gateway       Interface Priority HC-ID HC-Type HC-Result Description
	       192.168.193.0/24 0.0.0.0       test
	       192.168.212.0/24 192.168.193.1 test      10
	================================================================================
	```
- vlan
	```
	PAS-K(config)# show vlan
	
	================================================================================
	VLAN Configuration
	--------------------------------------------------------------------------------
	          |         |                                 |          |
	          |         |                   1 1 1 1 1 1 1 |          |
	VLAN Name | VLAN ID | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 | TBM      | Description
	--------------------------------------------------------------------------------
	default   | 1       | . u u u u u u u u u u u u u u u | disable  |
	test      | 193     | u . . . . . . . . . . . . . . . | disable  |
	================================================================================
	
	```
- H/C
	- TIP: 체크 할 터널 ip (vip)
	```
	PAS-K(config)# show health-check 1
	
	================================================================================
	  HEALTH-CHECK: 1
	 ------------------------------------------------------------------------------
	    ID                   : 1
	    Type                 : ip-tunnel-tcp
	    Timeout              : 3
	    Interval             : 5
	    Retry                : 3
	    Recover              : 0
	    Status               : enable
	    Graceful Shutdown    : disable
	    Description          :
	
	    --- Option ---------------------------------------------------------------
	    SIP                  :
	    TIP                  :
	    Inner TIP            : 192.168.193.50
	    DIP                  :
	    Port                 : 0
	================================================================================
	```
- real
	```
	PAS-K(config)# show real
	
	================================================================================
	REAL Configuration
	 ------------------------------------------------------------------------------
	  ID    Name   RIP             Rport SSL-Rport Weight Backup SVC-IP Status Description
	  1     server 192.168.211.150 80              1                    enable
	================================================================================
	```
- **port boundary 설정 (중요)**
	- 포트 바운더리 설정만 해주면 된다
	```
	================================================================================
	  PORT-BOUNDARY: 1
	 ------------------------------------------------------------------------------
	    ID                   : 1
	    Status               : enable
	    Type                 : include
	    Boundary             : all
	    Promisc              : off
	    Include MAC          : none
	    Port List            : 1,2
	    Vlan ID              :
	    Protocol             : all
	    Src IP               : 0.0.0.0/0
	    Dst IP               : 0.0.0.0/0
	    Src Port             :
	    Dst port             :
	    Description          :
	================================================================================
	```
### Server (ubuntu)
- ip tunnel 지원 확인
	```
	root@exam:/# modprobe ipip
	root@exam:/# lsmod | grep ipip
	ipip                   20480  0
	tunnel4                16384  1 ipip
	ip_tunnel              24576  1 ipip
	```
- 루프백 인터페이스에 vip 설정
	```
	root@exam:/# ip addr add 192.168.193.50/32 dev lo
	
	root@exam:/# ip addr
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	    inet 127.0.0.1/8 scope host lo
	       valid_lft forever preferred_lft forever
	    inet 192.168.193.50/32 scope global lo
	       valid_lft forever preferred_lft forever
	```
- vip 에 대해 arp 응답 하지 않도록 설정
	- lo인터페이스는 arp 응답을 하지는 않지만 전체 ip에 있으면 커널이 arp 응답을 할 수도 있기 때문에 설정
	```
	root@exam:/# sysctl -w net.ipv4.conf.all.arp_ignore=1
	net.ipv4.conf.all.arp_ignore = 1
	root@exam:/# sysctl -w net.ipv4.conf.all.arp_announce=2
	net.ipv4.conf.all.arp_announce = 2
	root@exam:/# sysctl -p
	```
- TCP 응답을 위해 rport(80)을 리스닝 할 nginx 설치
	```
	root@exam:/# apt install nginx -y
	
	root@exam:/# systemctl status nginx
	● nginx.service - A high performance web server and a reverse proxy server
	   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
	   Active: active (running) since Fri 2025-05-16 10:05:55 KST; 1min 16s ago

	root@exam:/# netstat -lntup grep :80
	Active Internet connections (only servers)
	Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
	tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      5439/nginx: master  
	tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      662/systemd-resolve 
	tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      957/sshd            
	tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      2285/cupsd          
	tcp6       0      0 :::80                   :::*                    LISTEN      5439/nginx: master  
	```
- [[리눅스 커널 기능|rp_filter]] 해제 
	- rp_filter는 패킷이 들어온 인터페이스로만 나가게 하는 기능 (보안 필터링 정책)
	- lo -> en1 로 통신 될 수 있도록 함
	```
	root@exam:/# cat /proc/sys/net/ipv4/conf/all/rp_filter
	1  ---> 현재 rp_filter 동작 중
	
	root@exam:/# sysctl -w net.ipv4.conf.all.rp_filter=0
	net.ipv4.conf.all.rp_filter = 0
	root@exam:/# sysctl -w net.ipv4.conf.default.rp_filter=0
	net.ipv4.conf.default.rp_filter = 0
	root@exam:/# 
	root@exam:/# cat /proc/sys/net/ipv4/conf/all/rp_filter
	0
	```
- 라우팅 정보
	```
	root@exam:/# ip addr
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	    inet 127.0.0.1/8 scope host lo
	       valid_lft forever preferred_lft forever
	    inet 192.168.193.50/32 scope global lo
	       valid_lft forever preferred_lft forever
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
	    link/ether d4:ae:52:ca:a2:98 brd ff:ff:ff:ff:ff:ff
	    inet 192.168.211.150/24 scope global eno1
	       valid_lft forever preferred_lft forever
	3: eno2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
	    link/ether d4:ae:52:ca:a2:99 brd ff:ff:ff:ff:ff:ff
	4: vmnet1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 1000
	    link/ether 00:50:56:c0:00:01 brd ff:ff:ff:ff:ff:ff
	    inet 172.16.12.1/24 brd 172.16.12.255 scope global vmnet1
	       valid_lft forever preferred_lft forever
	    inet6 fe80::250:56ff:fec0:1/64 scope link 
	       valid_lft forever preferred_lft forever
	5: vmnet8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 1000
	    link/ether 00:50:56:c0:00:08 brd ff:ff:ff:ff:ff:ff
	    inet 192.168.132.1/24 brd 192.168.132.255 scope global vmnet8
	       valid_lft forever preferred_lft forever
	    inet6 fe80::250:56ff:fec0:8/64 scope link 
	       valid_lft forever preferred_lft forever
	6: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
	    link/ipip 0.0.0.0 brd 0.0.0.0
	10: tun0@NONE: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1480 qdisc noqueue state UNKNOWN group default qlen 1000
	    link/ipip 192.168.211.150 peer 192.168.193.181
	    inet6 fe80::5efe:c0a8:d396/64 scope link 
	       valid_lft forever preferred_lft forever
	11: tun1@NONE: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 65515 qdisc noqueue state UNKNOWN group default qlen 1000
	    link/ipip 192.168.211.150 peer 192.168.193.50
	    inet6 fe80::5efe:c0a8:d396/64 scope link 
	       valid_lft forever preferred_lft forever
	root@exam:/# ip route
	default via 192.168.211.1 dev eno1 
	172.16.12.0/24 dev vmnet1 proto kernel scope link src 172.16.12.1 
	192.168.132.0/24 dev vmnet8 proto kernel scope link src 192.168.132.1 
	192.168.193.0/24 via 192.168.211.1 dev eno1 
	192.168.211.0/24 dev eno1 proto kernel scope link src 192.168.211.150 
	192.168.212.0/24 via 192.168.211.1 dev eno1 
	```
### 오류 확인
#### 오류_IPIP프로토콜 자동 미지원
- 커널에서 자동으로 IPIP프로토콜을 지원하지 않는 경우 (protocol 4 destination unreachable)
	```
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
	13:06:01.983492 00:06:c4:90:03:93 > d4:ae:52:ca:a2:98, ethertype IPv4 (0x0800), length 94: 10.10.10.1 > 10.10.10.10: 10.10.10.1.32623 > 192.168.193.50.80: Flags [S], seq 2115417017, win 64240, options [mss 1460,sackOK,TS val 658992584 ecr 0,nop,wscale 8], length 0 (ipip-proto-4)
	13:06:01.983539 d4:ae:52:ca:a2:98 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 122: 10.10.10.10 > 10.10.10.1: ICMP 10.10.10.10 protocol 4 port 60 unreachable, length 88
	```
- 해결
	- ipip 패킷을 받을 터널 인터페이스를 수동으로 생성
	- remote `[L4 인터페이스 ip]` local `[server real ip]`
		```
		root@exam:/# ip tunnel add tun0 mode ipip remote 192.168.193.181 local 192.168.211.150
		root@exam:/# ip link set tun0 up
		```
#### 오류_slb 안 됨 (hit 0)
- port boundary 설정 필요
- 증상 - tcpdump 확인 시 L4에서 서버로 패킷을 주지 않고 바로 차단
	```
	PAS-K(config)# tcpdump -nei test host 192.168.193.50
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on test, link-type EN10MB (Ethernet), capture size 65535 bytes
	09:38:24.351693 00:d0:f8:22:35:42 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 74: 192.168.193.50.80 > 192.168.193.181.56913: Flags [S.], seq 3079070524, ack 2850191002, win 65160, options [mss 1460,sackOK,TS val 2268603904 ecr 4085878557,nop,wscale 7], length 0
	09:38:24.352160 00:d0:f8:22:35:42 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 66: 192.168.193.50.80 > 192.168.193.181.56913: Flags [F.], seq 1, ack 2, win 510, options [nop,nop,TS val 2268603904 ecr 4085878557], length 0
	09:38:25.104383 00:06:c4:74:5b:b0 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 66: 192.168.212.225.50508 > 192.168.193.50.80: Flags [S], seq 276011400, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
	09:38:25.104429 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 54: 192.168.193.50.80 > 192.168.212.225.50508: Flags [R.], seq 0, ack 276011401, win 0, length 0
	09:38:25.608111 00:06:c4:74:5b:b0 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 66: 192.168.212.225.50508 > 192.168.193.50.80: Flags [S], seq 276011400, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
	```
## 결과(tcp)
### H/C 성공 확인
- PAS-K
	- 192.168.193.181(pask) > 192.168.211.150(rip)  :   192.168.193.181.55982 > 192.168.193.50.80  (outer : inner)
	- rip로 패킷을 보내서 실제 세션 (193.50:80) 이 vip로 리스닝 중인지 확인
		```
		PAS-K(config)# tcpdump -nei test host 192.168.193.50
		tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
		listening on test, link-type EN10MB (Ethernet), capture size 65535 bytes
		10:47:39.377309 00:d0:f8:22:35:42 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 74: 192.168.193.50.80 > 192.168.193.181.23114: Flags [S.], seq 749677678, ack 2096822829, win 65160, options [mss 1460,sackOK,TS val 2272758973 ecr 4090033583,nop,wscale 7], length 0
		10:47:39.377927 00:d0:f8:22:35:42 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 66: 192.168.193.50.80 > 192.168.193.181.23114: Flags [F.], seq 1, ack 2, win 510, options [nop,nop,TS val 2272758974 ecr 4090033583], length 0
		10:47:44.383902 00:d0:f8:22:35:42 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 74: 192.168.193.50.80 > 192.168.193.181.32664: Flags [S.], seq 285675813, ack 2483226022, win 65160, options [mss 1460,sackOK,TS val 2272763980 ecr 4090038589,nop,wscale 7], length 0
		10:47:44.384519 00:d0:f8:22:35:42 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 66: 192.168.193.50.80 > 192.168.193.181.32664: Flags [F.], seq 1, ack 2, win 510, options [nop,nop,TS val 2272763981 ecr 4090038590], length 0
		```
- server
	```
	root@exam:/# tcpdump -nei eno1 proto 4
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
	10:54:57.785909 00:d0:f8:22:35:42 > d4:ae:52:ca:a2:98, ethertype IPv4 (0x0800), length 94: 192.168.193.181 > 192.168.211.150: 192.168.193.181.55982 > 192.168.193.50.80: Flags [S], seq 2620729253, win 64240, options [mss 1460,sackOK,TS val 4090394018 ecr 0,nop,wscale 8], length 0 (ipip-proto-4)
	10:54:57.786072 00:d0:f8:22:35:42 > d4:ae:52:ca:a2:98, ethertype IPv4 (0x0800), length 86: 192.168.193.181 > 192.168.211.150: 192.168.193.181.55982 > 192.168.193.50.80: Flags [.], ack 2728183561, win 251, options [nop,nop,TS val 4090394018 ecr 2273119413], length 0 (ipip-proto-4)
	10:54:57.786316 00:d0:f8:22:35:42 > d4:ae:52:ca:a2:98, ethertype IPv4 (0x0800), length 86: 192.168.193.181 > 192.168.211.150: 192.168.193.181.55982 > 192.168.193.50.80: Flags [F.], seq 0, ack 1, win 251, options [nop,nop,TS val 4090394019 ecr 2273119413], length 0 (ipip-proto-4)
	10:54:57.786469 00:d0:f8:22:35:42 > d4:ae:52:ca:a2:98, ethertype IPv4 (0x0800), length 86: 192.168.193.181 > 192.168.211.150: 192.168.193.181.55982 > 192.168.193.50.80: Flags [.], ack 2, win 251, options [nop,nop,TS val 4090394019 ecr 2273119413], length 0 (ipip-proto-4)
	10:55:02.791236 00:d0:f8:22:35:42 > d4:ae:52:ca:a2:98, ethertype IPv4 (0x0800), length 94: 192.168.193.181 > 192.168.211.150: 192.168.193.181.56986 > 192.168.193.50.80: Flags [S], seq 4008148832, win 64240, options [mss 1460,sackOK,TS val 4090399023 ecr 0,nop,wscale 8], length 0 (ipip-proto-4)
	```
- 응답 - vip로 서버에게 응답
	```
	root@exam:/# tcpdump -nei eno1 host 192.168.193.50
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
	11:06:33.639879 d4:ae:52:ca:a2:98 > 00:00:5e:00:01:d3, ethertype IPv4 (0x0800), length 74: 192.168.193.50.80 > 192.168.193.181.35591: Flags [S.], seq 2636425598, ack 3869140415, win 65160, options [mss 1460,sackOK,TS val 2273815267 ecr 4091089864,nop,wscale 7], length 0
	11:06:33.640367 d4:ae:52:ca:a2:98 > 00:00:5e:00:01:d3, ethertype IPv4 (0x0800), length 66: 192.168.193.50.80 > 192.168.193.181.35591: Flags [F.], seq 1, ack 2, win 510, options [nop,nop,TS val 2273815267 ecr 4091089865], length 0
	```
#### 리스닝 포트(80) 닫은 경우
- 서버가 reset 전송
	```
	17:58:20.629904 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 94: 192.168.193.181 > 192.168.211.150: 192.168.193.181.60971 > 192.168.193.50.80: Flags [S], seq 4157529438, win 64240, options [mss 1460,sackOK,TS val 4115874836 ecr 0,nop,wscale 8], length 0 (ipip-proto-4)
	17:58:20.630125 00:d0:f8:22:35:42 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 60: 192.168.193.50.80 > 192.168.193.181.60971: Flags [R.], seq 0, ack 4157529439, win 0, length 0
	```
### SLB 확인
#### slb 결과 확인
```
PAS-K(config)# show info slb test

================================================================================
  SLB: test
 ------------------------------------------------------------------------------
    Name                 : test
    IP Version           : ipv4
    Status               : enable
    Priority             : 50
    NAT Mode             : l3dsr-iptunnel
    LB Method            : rr
    Fail Skip            : none
    Fail Action          : default
    Session Timeout Mode : global
    Session Reset        : none
    Session-sync         : none
    H/C Condition        : all
    Health Check         : 1
    Passive Health Check :
    Service Health       : ACT

    Vip
       VIP            Protocol Vport
       192.168.193.50 tcp      80

    Description          :

    Filter
       ID    Type    Protocol Src IP    Dst IP            Status Hit   Description
       1     include tcp      0.0.0.0/0 192.168.193.50/32 enable 1

    Sticky
        Time             : 60

        --- Option -----------------------------------------------------------
        Src Subnet       : 255.255.255.255
        Dst Subnet       : 0.0.0.0

    Keep-Backup
        Service          : disable
        Real             : disable

    Backup               :

    Dynamic-Filter       :

    Real
       ID    Name   RIP             Rport Backup Status G-SHDN  Health Cause State-Time      Description
       1     server 192.168.211.150 80           enable disable ACT          0 days 17:27:32

    Health-Check-Info
       ID    Type          Port  Status
       1     ip-tunnel-tcp 0     enable

    Health-Check-Result
        ID Name   Total Active: 1
        1  server     O         O (0 ms)

    Real Health-Check-Result
        ID Total
        1      D

    Statistics
        Real
           ID    Name   Current sessions Total sessions
           1     server 0                1

        CPS              : 0
        Current sessions : 0
        Total sessions   : 1

    Service-Chain-Info   :

================================================================================
```
- show entry
	```
	PAS-K# show entry
	================================================================================
	Prot [Org]Sip:Sport        Dip:Dport         - [Rep]Sip:Sport    Dip:Dport             Svc:Real   [R]Svc:Real
	--------------------------------------------------------------------------------
	tcp  192.168.212.225:54136 192.168.193.50:80 - 192.168.193.50:80 192.168.212.225:54136 slb.test:1
	================================================================================
	```
#### tcpdump 확인
- pask (클라이언트가 L4로 요청하는 패킷)
	- 192.168.212.225.51186 > 192.168.193.50.80: Flags [S] ->클라이언트 요청
	- 192.168.212.225.51186 > 192.168.193.50.80: Flags [P.] -> http 응답
		```
		PAS-K# tcpdump -nei test host 192.168.212.225
		tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
		listening on test, link-type EN10MB (Ethernet), capture size 65535 bytes
		12:36:10.851276 00:06:c4:74:5b:b0 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 66: 192.168.212.225.52316 > 192.168.193.50.80: Flags [S], seq 1974408078, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
		12:36:10.869031 00:06:c4:74:5b:b0 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 132: 192.168.212.225.52316 > 192.168.193.50.80: Flags [P.], seq 1974408079:1974408157, ack 3169393736, win 255, length 78
		12:36:10.886833 00:06:c4:74:5b:b0 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 60: 192.168.212.225.52316 > 192.168.193.50.80: Flags [F.], seq 78, ack 860, win 252, length 0
		12:36:10.887529 00:06:c4:74:5b:b0 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 60: 192.168.212.225.52316 > 192.168.193.50.80: Flags [.], ack 861, win 252, length 0	
		```
- pask (L4가 클라이언트로부터 온 패킷을 받아서 서버의 rip로 넘겨주는 패킷)
	- 192.168.193.181 > 192.168.211.150: 192.168.212.225.52446 > 192.168.193.50.80: Flags [S] //l4 인터페이스에서 서버 rip로 전달 (내부에는 클라이언트 ip > vip) 
	```
	PAS-K# tcpdump -nei test host 192.168.193.181
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on test, link-type EN10MB (Ethernet), capture size 65535 bytes
	12:50:22.287400 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 86: 192.168.193.181 > 192.168.211.150: 192.168.212.225.52446 > 192.168.193.50.80: Flags [S], seq 1236981390, win 65535, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0 (ipip-proto-4)
	12:50:22.288155 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 74: 192.168.193.181 > 192.168.211.150: 192.168.212.225.52446 > 192.168.193.50.80: Flags [.], ack 347682253, win 255, length 0 (ipip-proto-4)
	12:50:22.290883 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 152: 192.168.193.181 > 192.168.211.150: 192.168.212.225.52446 > 192.168.193.50.80: Flags [P.], seq 0:78, ack 1, win 255, length 78 (ipip-proto-4)
	12:50:22.306162 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 74: 192.168.193.181 > 192.168.211.150: 192.168.212.225.52446 > 192.168.193.50.80: Flags [F.], seq 78, ack 860, win 252, length 0 (ipip-proto-4)
	12:50:22.306789 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 74: 192.168.193.181 > 192.168.211.150: 192.168.212.225.52446 > 192.168.193.50.80: Flags [.], ack 861, win 252, length 0 (ipip-proto-4)
	12:50:23.292305 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 94: 192.168.193.181 > 192.168.211.150: 192.168.193.181.64851 > 192.168.193.50.80: Flags [S], seq 2325090198, win 64240, options [mss 1460,sackOK,TS val 4097397498 ecr 0,nop,wscale 8], length 0 (ipip-proto-4)
	12:50:23.292535 00:d0:f8:22:35:42 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 74: 192.168.193.50.80 > 192.168.193.181.64851: Flags [S.], seq 2727981331, ack 2325090199, win 65160, options [mss 1460,sackOK,TS val 2280122968 ecr 4097397498,nop,wscale 7], length 0
	12:50:23.292570 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 86: 192.168.193.181 > 192.168.211.150: 192.168.193.181.64851 > 192.168.193.50.80: Flags [.], ack 1, win 251, options [nop,nop,TS val 4097397498 ecr 2280122968], length 0 (ipip-proto-4)
	12:50:23.292826 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 86: 192.168.193.181 > 192.168.211.150: 192.168.193.181.64851 > 192.168.193.50.80: Flags [F.], seq 1, ack 1, win 251, options [nop,nop,TS val 4097397499 ecr 2280122968], length 0 (ipip-proto-4)
	12:50:23.293061 00:d0:f8:22:35:42 > 00:06:c4:90:03:93, ethertype IPv4 (0x0800), length 66: 192.168.193.50.80 > 192.168.193.181.64851: Flags [F.], seq 1, ack 2, win 510, options [nop,nop,TS val 2280122969 ecr 4097397499], length 0
	12:50:23.293080 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 86: 192.168.193.181 > 192.168.211.150: 192.168.193.181.64851 > 192.168.193.50.80: Flags [.], ack 2, win 251, options [nop,nop,TS val 4097397499 ecr 2280122969], length 0 (ipip-proto-4)
	```
- 서버
	- 192.168.193.50.80 > 192.168.212.225.52248: Flags [P.] (서버가 src ip를 vip로 클라이언트에게 직접 응답 확인)
	- 00:00:5e:00:01:d3은 GW mac
	```
	root@exam:/# tcpdump -nei eno1 host 192.168.193.50
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
	12:32:27.949041 d4:ae:52:ca:a2:98 > 00:00:5e:00:01:d3, ethertype IPv4 (0x0800), length 66: 192.168.193.50.80 > 192.168.212.225.52248: Flags [S.], seq 3950682216, ack 2999346906, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
	12:32:27.965671 d4:ae:52:ca:a2:98 > 00:00:5e:00:01:d3, ethertype IPv4 (0x0800), length 54: 192.168.193.50.80 > 192.168.212.225.52248: Flags [.], ack 79, win 502, length 0
	12:32:27.965922 d4:ae:52:ca:a2:98 > 00:00:5e:00:01:d3, ethertype IPv4 (0x0800), length 913: 192.168.193.50.80 > 192.168.212.225.52248: Flags [P.], seq 1:860, ack 79, win 502, length 859: HTTP: HTTP/1.1 200 OK
	12:32:27.981597 d4:ae:52:ca:a2:98 > 00:00:5e:00:01:d3, ethertype IPv4 (0x0800), length 54: 192.168.193.50.80 > 192.168.212.225.52248: Flags [F.], seq 860, ack 80, win 502, length 0
	```
## 구성/결과(udp)
### 서버
- udp 포트에 대해 응답 할 snmp (161)패키지 설정
	- apt install -y snmpd
- 외부 ip에서도 수신 가능하도록 설정
	```
	vi /etc/snmp/snmpd.conf
	-> agentAddress  udp:161 로 수정
	root@exam:/# systemctl restart snmpd
	root@exam:/# netstat -lntup
	Active Internet connections (only servers)
	Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
	tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      24759/nginx: master 
	tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      21203/systemd-resol 
	tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      957/sshd            
	tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      23130/cupsd         
	tcp6       0      0 :::22                   :::*                    LISTEN      957/sshd            
	tcp6       0      0 ::1:3350                :::*                    LISTEN      937/xrdp-sesman     
	tcp6       0      0 ::1:631                 :::*                    LISTEN      23130/cupsd         
	tcp6       0      0 :::23                   :::*                    LISTEN      1169/xinetd         
	tcp6       0      0 :::3389                 :::*                    LISTEN      959/xrdp            
	udp        0      0 0.0.0.0:40326           0.0.0.0:*                           26170/snmpd         
	udp        0      0 127.0.0.53:53           0.0.0.0:*                           21203/systemd-resol 
	udp        0      0 0.0.0.0:161             0.0.0.0:*                           26170/snmpd         
	udp        0      0 0.0.0.0:631             0.0.0.0:*                           23131/cups-browsed  
	udp        0      0 0.0.0.0:46204           0.0.0.0:*                           814/avahi-daemon: r 
	udp        0      0 0.0.0.0:5353            0.0.0.0:*                           814/avahi-daemon: r 
	udp6       0      0 :::59934                :::*                                814/avahi-daemon: r 
	udp6       0      0 :::5353                 :::*                                814/avahi-daemon: r 
	```
### PAS-K
- ip-tunnel-udp 헬스체크 생성
	```
	PAS-K(config)# show health-check 2
	
	================================================================================
	  HEALTH-CHECK: 2
	 ------------------------------------------------------------------------------
	    ID                   : 2
	    Type                 : ip-tunnel-udp
	    Timeout              : 3
	    Interval             : 5
	    Retry                : 3
	    Recover              : 0
	    Status               : enable
	    Graceful Shutdown    : disable
	    Description          :
	
	    --- Option ---------------------------------------------------------------
	    SIP                  :
	    TIP                  :
	    Inner TIP            : 192.168.193.50
	    DIP                  :
	    Port                 : 161
	================================================================================
	```
### 결과 확인
- pas-k에서 udp 161로 보내는 헬스체크 패킷 확인
- **헬스체크 패킷인 udp는 ipip로 감싸져 있기 때문에 port udp로 필터링 해서 보면 보이지 않음**
	```
	PAS-K(config)# tcpdump -nei test proto 4
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on test, link-type EN10MB (Ethernet), capture size 65535 bytes
	13:55:55.763758 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 62: 192.168.193.181 > 192.168.211.150: 192.168.193.181.60101 > 192.168.193.50.161:  [nothing to parse] (ipip-proto-4)
	13:56:00.769031 00:06:c4:90:03:93 > 00:00:5e:00:01:c1, ethertype IPv4 (0x0800), length 62: 192.168.193.181 > 192.168.211.150: 192.168.193.181.34308 > 192.168.193.50.161:  [nothing to parse] (ipip-proto-4)
	```
