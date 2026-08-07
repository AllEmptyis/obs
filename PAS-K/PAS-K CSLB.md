# 주의
- slb vip로 구성하는 경우 real 

# pas-k에 nat 테이블 넣는 방법
- nat 테이블을 넣어야 하는 이유
```
nat를 하지 않으면 캐시서버로부터 동일한 세션이 그대로 pask로 돌아오게 된다
즉 서로 주고 받다가 pask가 다시 클라이언트한테 리셋을 보냄

cslb 리버스 엔트리는 들어온 smac와 real mac을 비교해서 동일한 경우 생성
그러나 먼저 pask는 생성되어 있는 포워드 엔트리와 비교하여 동일한 경우 그 포워드 엔트리 세션이라고 생각함

그래서 캐시 서버에서 5tuple 중 (mac 제외, 생성되어 있는 포워드 엔트리와 비교할 수 있는 것) 한가지라도 바꿔서 다른 세션인 걸로 보내줘야 pask가 그 때 smac-real mac을 비교하게 됨

추가:
이후 slb를 통해 서버를 거쳐서 다시 캐시 서버로 오게 되면 그 때 캐시 서버가 자동으로 nat했던 세션을 원복해준다
(리눅스 내부의 conntrack을 통해 관리함)
세션을 원복해줘야 기존 클라이언트와 통신할 수 있음
```
- 명령어
	- shell에서 `ns-svc` 진입
	- iptables -t nat -A POSTROUTING -p tcp -j SNAT --to-source :600-700
```
root@cache(svc_net):~# iptables -t nat -L
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DNAT       all  -- !br0                  anywhere             connmark match  0x726 to:203.0.113.100

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  anywhere            !br0                  connmark match  0x726
SNAT       tcp  --  anywhere             anywhere             to::600-700
```

