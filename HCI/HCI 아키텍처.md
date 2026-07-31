# 아키텍처 구성
- 구성
	- vswitch = bridge (오픈스택 네트워크) 와 같은 역할
```
[VM] -- vNIC -- [Virtual Switch] -- [bond] -- pNIC -- [Physical Switch]
```
## virtual switch
- 주요 기능
	- vm의 nic을 호스트의 물리 네트워크에 연결
	- vm간 트래픽 스위칭/라우팅
	- vxlan 등을 사용해서 오버레이 트래픽 관리


# hyperthreading
- intel CPU의 SMT(Simultaneous Multi Threasing) 기술
- 물리 코어 1개가 논리적으로 2개의 스레드(코어)처럼 동작하도록 하는 기술
	- ex) 물리 코어 16개 -> 32개로 인식
- 실제 연산 유닛이 늘어나는 것은 아니고 하나의 코어 안에서 스레드가 번갈아가면서 실행됨
	- 즉 2배 효과는 아님

# overcommit
- 물리적으로 실제하지 않는 vcpu를 가상으로 더 할당하는 개념
- 오픈스택 노바 - cpu_allocation_ratio 설정값과 동일
	- ex) 논리코어 32개, 오버커밋 8 -> 256개로 계산
- 사용 이유: 대부분의 vm이 cpu를 100% 사용하지 않기 때문
	- 단, vm이 자원을 100% 사용하게 되면 성능 저하 발생