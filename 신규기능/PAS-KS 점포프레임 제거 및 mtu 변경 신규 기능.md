# 일감
- https://redmine.piolink.com/issues/166741

# 내용 요약
- 클라우드 환경에서는 하이퍼바이저별로 다양한 네트워크 드라이버를 사용함
- 기존처럼 점보프레임 사용- 9200 mtu로 설정하는 방식은 클라우드 방식에 맞지 않음
	- 네트워크 드라이버별로 지원하는 max mtu가 다르다
- 따라서 기존 점보프레임 기능은 제거, 드라이버 별 max mtu 값을 포트의 mtu로 초기화 (부팅 시)
- 용어 정리
	- 포트: vnic에 연결되는 가상 포트 (포트의 mtu가 초기화 됨)
	- 인터페이스: 보통 인터페이스에 포트를 할당해서 사용함
	  ks에서 포트를 받았을 때는 인터페이스 기준으로 처리함
	  그렇기 때문에 이 기능은 클라우드 환경에 맞게 (어차피 최대 mtu는 그 드라이버의 max mtu) 인터페이스의 mtu를 수동으로 설정하여 사용하는 기능
# vmware
- 기본 mtu
	- mgmt 1500 (고정)
	- eth1: 16110
	- test에 할당한 eth1: 1500
```
3: mgmt: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:6e:38:9f brd ff:ff:ff:ff:ff:ff
    inet 192.168.212.63/24 brd 192.168.212.255 scope global mgmt
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe6e:389f/64 scope link
       valid_lft forever preferred_lft forever
4: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 16110 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:6e:38:a9 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::20c:29ff:fe6e:38a9/64 scope link
       valid_lft forever preferred_lft forever
5: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
6: _ibh@bond0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 82:36:b9:ba:d4:65 brd ff:ff:ff:ff:ff:ff
7: _tbm@bond0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 82:36:b9:ba:d4:65 brd ff:ff:ff:ff:ff:ff
9: test@eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:0c:29:6e:38:a9 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::20c:29ff:fe6e:38a9/64 scope link
       valid_lft forever preferred_lft forever
```


부팅했을 때 포트 mtu로 초기화 되는지? (네트워크 드라이버 별 확인)
- e1000: 16110, virtualbox, vmare -> 확인
- vmnet3: 9000, vmware
- virtio: 오픈스택, 가변 -> 확인
mtu 상한선이 포트 mtu로 맞춰졌는지? 
클라우드 환경에서 mtu 변경 시 동일하게 맞춰지는지?
인터페이스가 할당된 vlan mtu가 인터페이스 mtu로 맞춰지는지?