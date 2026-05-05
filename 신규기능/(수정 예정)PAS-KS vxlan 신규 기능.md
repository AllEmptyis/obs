# 일감
- https://redmine.piolink.com/issues/129448
# 테스트 환경
- 192.168.211.100 서버에 ks 설치해서 구성
- kvm 설치 명령어
```
virt-install --name PASK-KS-v2.2.7.1.1 --memory 8012 --vcpu 2 --disk path=/home/PLOS-PASK-KS-v2.2.7.1.1.qcow2 --network bridge=br0 --import --os-variant generic
```
- 설치가 되면 dhcp로 mgmt ip를 자동으로 받아온다

# 테스트 환경
- 오픈스택 외부망에 pas-ks를 올리고, 내부망에 있는 vm으로 slb 예정
	- pas-ks가 vtep 역할을 하여 vxlan 동작하는지 확인할 예정
- 오픈스택에서 vxlan 사용하도록 설정 변경 필요
	- /etc/neutron/plugins/ml2/ml2_conf.ini
```
[ml2]

#
# From neutron.ml2
#

# List of network type driver entrypoints to be loaded from the
# neutron.ml2.type_drivers namespace. (list value)
#type_drivers = local,flat,vlan,gre,vxlan,geneve
type_drivers=flat,vxlan
tenant_network_types = vxlan <- 추가



[ml2_type_vxlan]

#
# From neutron.ml2
#

# Comma-separated list of <vni_min>:<vni_max> tuples enumerating ranges of
# VXLAN VNI IDs that are available for tenant network allocation (list value)
#vni_ranges =

# Multicast group for VXLAN. When configured, will enable sending all broadcast
# traffic to this multicast group. When left unconfigured, will disable
# multicast VXLAN mode. (string value)
#vxlan_group = <None>
vni_ranges = 1:1000 <-레인지 추가
```
- 현재 ovn이 사용 중인 encap type 변경
```
ovs-vsctl get Open_vSwitch . external_ids:ovn-encap-type 
//현재 사용 중인 타입 확인
ovs-vsctl set Open_vSwitch . external_ids:ovn-encap-type=vxlan
//vxlan으로 변경
```
- 데몬 재시작
```
systemctl restart neutron-server
systemctl restart ovn-controller
```
- 새 네트워크 생성해야 vxlan으로 생성됨
	- gui에서 생성하면 기본 제네브로 잡기 때문에 cli에서 vxlan 타입 지정해서 생성
```
[root@openstack ml2(keystone_admin)]# openstack network create --provider-network-type vxlan test-vxlan
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2026-04-07T08:51:22Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 1490321c-9039-44ba-baa4-63392dd1628d |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | test-vxlan                           |
| port_security_enabled     | True                                 |
| project_id                | a309d0dc5ca54274aaa6558e50ac1fbb     |
| provider:network_type     | vxlan                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 10                                   |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| tenant_id                 | a309d0dc5ca54274aaa6558e50ac1fbb     |
| updated_at                | 2026-04-07T08:51:22Z                 |
+---------------------------+--------------------------------------+
```
- qcow2 이미지를 가지고 인스턴스를 생성한다
## 네트워크 구성
- 생성 순서
	- 내부망, 외부망 생성 후 라우터 생성(내부망 인터페이스 추가, external network는 public으로 해야 외부랑 통신 됨)
	- 외부망 생성 시 gw는 실제 통신하는 gw로 잡을 것
```
내부망(11.11.11.1)- 라우터(vxlan_router) - 프로바이더(공유망)
pas-ks: 11.11.11.57
라우터 외부 인터페이스(external gw ip): snat 되는 ip, 192.168.211.218
//external gw는 내부의 가상 라우터가 외부망에서 자신을 식별하기 위한 주소
```
- 통신 할 때, 내부망에 있는 vm은 external gw ip(라우터 외부 인터페이스)로 snat 해서 외부와 통신하며, 외부에서 직접 vm에 접근은 불가
- 외부망에 있는 vm은 가상라우터를 거치지 않고 통신한다


