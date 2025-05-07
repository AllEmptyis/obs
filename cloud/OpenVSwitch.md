## 개념
- 리눅스에서 사용하는 가상 네트워크 스위치
## Linux Bridge
### 정의
- 리눅스 커널에 기본 내장
- 주로 L2 기반 간단한 네트워크 구성에 사용
### 구조
- 브릿지 커널 모듈은 mac learning 기능이 있으며, mac table은 커널 메모리에 존재
	- bridge fdb 로 확인
#### 구성
bridge.ko: 브릿지의 커널 모듈
br0 : 각 브릿지의 인터페이스
- 내부 MAC table을 가지고 트래픽 전달
```
+-----------+        +-----------+
|   tap0    |        |   eth0    |
| (VM NIC)  |        | (Host NIC)|
+-----------+        +-----------+
       \              /
      +--------------+
      |  br0 (bridge)|
      +--------------+
```

### 주요 기능
- 기본 L2 스위칭
- VLAN (802.1Q 태깅)
- 트렁크 포트 -> 제한적
### 관리 명령어
- nmcli (구: ip link, brctl)
## OpenVswitch
### 정의
- 고급 기능이 포함 된 가상 스위치 소프트웨어 (패키지 설치 필요)
- 데몬 + 커널 형태로 동작
- L3, VLAN, SDN, 멀티 테넌시 구성 등
### 구조
- 하이브리드
	- fast path=커널
	- slow path=데몬
- 커널 모듈
	- **openvswitch.ko : 실제 패킷 처리**
	- vxlan (선택)
	- gre, geneve (선택): 터널링
	- nf_conntrack
- 브리지
	- br-ex: 실제 물리 NIC으로 외부 네트워크와 연결
	- br-int: 내부 브릿지
- 유저공간 데몬도 함께 작동 (ovs-vswitchd, ovsdb-server)
```
+-----------+     +-----------+     +-------------+
|   tap0    |     |   eth0    |     |  vxlan0     |
+-----------+     +-----------+     +-------------+
       \              |                   /
        \             |                  /
         +------------------------------+
         |     br-int/ex (OVS bridge)      |
         +------------------------------+
                   |
              ovs-vswitchd (rule)
                   |
            ovsdb-server (관리)
```
### 주요 기능
- 기본 L2 브릿지
- VLAN
- VXLAN
- QoS
- OpenFlow / SDN 연동 (SDN에서 데이터 플레인 역할)
- 포트 미러링
### 설치 및 구성
- 패키지 설치
#### 관리 명령어
- ovs-vsctl, ovs-ofctl
#### 데몬
- ovs-vswitchd : 브릿지로 들어오는 패킷이 어디로 갈지 판단 (룰)
- ovsdb-server : 구성 정보 관리 (포트, 브리지)
### 사용
- 오픈스택, 쿠버네티스 등에서 주로 사용 (overlay 구성)
- SDN 환경