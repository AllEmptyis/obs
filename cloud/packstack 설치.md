# 설치 환경
- 192.168.211.100 서버 위에 VM 생성
	- 192.168.211.106 static ip 생성
- rocky 9.7 minimal
	- 실제로는 gui도 설치됨
	- 로키 9 이상 packstack 공식 지원
- 스펙
	- 메모리 24GB, vcpu 8
	- 디스크 100GB
- 설치 명령어
```
virt-install --name packstack --memory 24000 --cdrom /home/Rocky-9.6-x86_64-minimal.iso --osinfo detect=on,require=off --network bridge=br0 --disk path=/home/packstack.qcow2,size=100 --vcpus 8
```
- 공식 문서
	- https://docs.openstack.org/install-guide/environment-packages-rdo.html?utm_source=chatgpt.com

# 환경 구성
## 호스트 네임 설정
- 오픈스택은 각 노드별로 호스트네임 기반으로 통신
	- DNS에 각 노드별 호스트네임이 등록되어 있음
	- 따라서 /etc/hosts에 등록 필요 없음
- packstack은 내부적으로 멀티노드가 있는 것처럼 동작, 각 노드는 동일한 호스트네임으로 통신함
	- /etc/hosts에 호스트네임 등록 필요 (dns 대용)
	- alias도 등록 필요 
```
hostnamectl set-hostname openstack.test.com
echo "192.168.211.106 openstack.test.com openstack" | sudo tee -a /etc/hosts
```
## 방화벽 해제
```
/etc/selinux/config
SELINUX=permissiv 변경

setenforce 0

systemctl disable --now firewalld
```
## 포워딩 설정
- 오픈스택에서도 패킷 포워딩 할 수 있도록 영구 설정
```
/etc/sysctl.conf에 추가
net.ipv4.ip_forward = 1

sysctl -p //적용
```
## 네트워크 구성 방법
- rocky 환경에서는 네트워크 매니저 활성화 필요
- 구성 요약
	- br-ex: 네트워크 스위치 본체, 모든 데이터 모이는 중심
	- br-ex-enp1s0-port, enp1s0:  스위치의 1번 포트. 실제 ip를 가지고 통신하지 않기 때문에 ipv4.method disabled로 설정
	- br-ex-por, br-ex-ioface: 스위치의 2번 포트. 운영체제가 사용하는 가상 랜카드, 여기에 실제 통신할 ip 부여
```
br-ex (ovs-bridge)
  └── br-ex-port (ovs-port)         ← br-ex-iface의 master (IP 할당용)
        └── br-ex-iface (ovs-interface, IP)
  └── br-ex-enp1s0-port (ovs-port)  ← enp1s0의 포트
        └── br-ex-enp1s0 (ethernet)
```
### 구성 명령어
- conn.interface는 실제 커널에서 보여지는 장치 이름
- con-name은 nm이 관리하는 이름 (자유롭게 지정 가능)
```
#!/bin/bash

# 1. 기존에 꼬여있을 수 있는 동일 이름의 커넥션 삭제 (클린 설치용)
nmcli conn del br-ex br-ex-port br-ex-iface br-ex-enp1s0-port br-ex-enp1s0 enp1s0 2>/dev/null

# 2. OVS 브릿지 생성
nmcli conn add type ovs-bridge conn.interface br-ex con-name br-ex

# 3. OVS 내부 인터페이스용 포트 생성
nmcli conn add type ovs-port conn.interface br-ex-port master br-ex con-name br-ex-port

# 4. 실제 IP가 할당될 내부 인터페이스 생성 (Manual 설정)
nmcli conn add type ovs-interface slave-type ovs-port conn.interface br-ex-iface master br-ex-port con-name br-ex-iface \
ipv4.method manual ipv4.addr 192.168.211.106/24 ipv4.gateway 192.168.211.1 ipv4.dns 8.8.8.8

# 5. 물리 NIC용 OVS 포트 생성
nmcli conn add type ovs-port conn.interface br-ex-enp1s0-port master br-ex con-name br-ex-enp1s0-port

# 6. 물리 NIC(enp1s0)를 포트에 바인딩 (여기가 핵심: IP 기능을 완전히 끕니다)
nmcli conn add type ethernet conn.interface enp1s0 master br-ex-enp1s0-port con-name br-ex-enp1s0 \
ipv4.method disabled ipv6.method disabled

# 7. 활성화 (안정성을 위해 브릿지부터 순서대로)
nmcli conn up br-ex
nmcli conn up br-ex-port
nmcli conn up br-ex-iface
nmcli conn up br-ex-enp1s0-port
nmcli conn up br-ex-enp1s0
```
### 구성 확인 (완료)
```
[root@openstack ~]# nmcli conn show
NAME               UUID                                  TYPE           DEVICE
br-ex-iface        8d752eae-ed67-43f2-9858-6c590a35a964  ovs-interface  br-ex-iface
br-ex              8d87d6a4-8a63-4dac-87cf-5c682a3593bd  ovs-bridge     br-ex
br-ex-enp1s0       acee5779-5afc-4257-97fb-390ef62c2318  ethernet       enp1s0
br-ex-enp1s0-port  8dbfd6ef-76fb-43cf-89d1-16e3101b40ea  ovs-port       br-ex-enp1s0-port
br-ex-port         66831f7a-0b45-43de-81d9-39b561562045  ovs-port       br-ex-port
lo                 0f0173ba-c1b2-42d6-8010-f01579cc702b  loopback       lo


[root@openstack ~]# nmcli device
DEVICE                                                        TYPE           STATE          CONNECTION
br-ex-iface                                                   ovs-interface  연결됨         br-ex-iface
enp1s0                                                        ethernet       연결됨         br-ex-enp1s0
br-ex                                                         ovs-bridge     연결됨         br-ex
br-ex-enp1s0-port                                             ovs-port       연결됨         br-ex-enp1s0-port
br-ex-port                                                    ovs-port       연결됨         br-ex-port
lo                                                            loopback       연결됨 (외부)  lo
patch-br-int-to-provnet-821d45b7-581f-41ba-a1a7-685e63eda66f  ovs-interface  연결 끊겼음    --
patch-provnet-821d45b7-581f-41ba-a1a7-685e63eda66f-to-br-int  ovs-interface  연결 끊겼음    --
br-int                                                        openvswitch    관리되지 않음  --
ovs-system                                                    openvswitch    관리되지 않음  --
br-int                                                        ovs-bridge     관리되지 않음  --
br-int                                                        ovs-port       관리되지 않음  --
patch-br-int-to-provnet-821d45b7-581f-41ba-a1a7-685e63eda66f  ovs-port       관리되지 않음  --
patch-provnet-821d45b7-581f-41ba-a1a7-685e63eda66f-to-br-int  ovs-port       관리되지 않음  --
tap341d603a-b5                                                ovs-port       관리되지 않음  --
tapcc1631c7-f0                                                ovs-port       관리되지 않음  --
```
### bridge mapping 설정
- bridge mapping은 오픈스택 내부에 생성된 vm이 인식하는 ovs 브릿지 이름을 지정하는 것
- 현재 백엔드로 ovn 사용 중이기 때문에 cli에서 아래 명령어 입력
```
ovs-vsctl set open . external-ids:ovn-bridge-mappings=physnet1:br-ex

의미: 
ovs-vsctl: open vswitch의 설정 DB를 조회/수정하는 명령어
set open . : 현재 시스템 수정하겠다는 의미
external-ids: 메타데이터
ovn-bridge-mappings: OVN 에이전트 키 값
physnet1:br-ex
-> physnet1: 별칭
br-ex: 실제 이름
```
- /etc/neutron/plugins/ml2/ml2_conf.ini 확인
```
[ml2_type_flat]

#
# From neutron.ml2
#

# List of physical_network names with which flat networks can be created. Use
# default '*' to allow flat networks with arbitrary physical_network names. Use
# an empty list to disable flat networks. (list value)
#flat_networks = *
flat_networks=*
//혹은 * 부분에 physnet1으로 지정
```
### ovs와 리눅스 브릿지 차이
- ovs (openvswitch)
	- 뉴트론이 관리, neutron ML2 플러그인으로 제어
	- 오픈스택 vm 트래픽
	- ovs-vsctl로 설정
- 리눅스 브릿지
	- 커널 내장, neutron ML2 Linux bridge 드라이버로 제어
	- 일반 네트워크 브릿지
	- brctl, nmcli로 설정
## 패키지 / 오픈스택 설치
```
dnf config-manager --enable crb
dnf install -y centos-release-openstack-caracal
dnf -y install openvswitch
// 밑에 오픈스택 설치 전 브릿지 인터페이스 사전 구성 때문에 필요

# Packstack 설치
dnf install -y openstack-packstack
```
- 설치 확인
```
[root@localhost ~]# packstack --version
packstack 24.0.0
```
- openvswitch 설치/활성화
```
dnf -y install openvswitch
systemctl enable --now openvswitch
```
- answer file 생성
	- 해당 파일을 기반으로 오픈스택 설치됨
```
[root@localhost ~]# packstack --gen-answer-file=/root/packstack-answer.txt

Packstack changed given value to required value /root/.ssh/id_rsa.pub
-> ssh 공개키 경로 자동 생성
오픈스택은 노드간 ssh 접속 + ssh key 기반 인증 사용

Additional information:
 * Parameter CONFIG_NEUTRON_L2_AGENT: You have chosen OVN Neutron backend. Note that this backend does not support the VPNaaS plugin. Geneve will be used as the encapsulation method for tenant networks
```
- answerfile 확인/수정 내용
```
CONFIG_KEYSTONE_ADMIN_PW=twoneulbo0510! <-패스워드 변경
CONFIG_NEUTRON_L3_EXT_BRIDGE=br-ex
CONFIG_CONTROLLER_HOST=192.168.211.106
CONFIG_COMPUTE_HOSTS=192.168.211.106
CONFIG_HORIZON_INSTALL=y

CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-ex:enp1s0
CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=extnet:br-ex
CONFIG_NEUTRON_OVS_TUNNEL_IF=192.168.211.106

Neutron ML2 driver: 뉴트론의 네트워크 요청을 OVN이 알아들을 수 있도록 해주는 드라이버
br-ex는 설치되면서 자동으로 생성됨
리눅스 브릿지 굳이 만들 필요 없음

//우선 끌 것
CONFIG_CINDER_INSTALL=n
CONFIG_SWIFT_INSTALL=n

CONFIG_MARIADB_PW=twoneulbo0510!
CONFIG_NOVA_DB_PW=twoneulbo0510!
CONFIG_NTP_SERVERS=pool.ntp.org
```
- answer file 기반으로 오픈스택 설치 진행
	-  packstack --allinone
	- packstack --answer-file=/root/packstack-answer.txt
		- answerfile 지정해서 설치 
## 설치 중 DB 꼬일 경우
- packstack은 mariaDB에 db 계정 1개+DB 3개 생성
- 최종적으로 아래와 같이 되어야 함
```
MariaDB [(none)]> SELECT user, host FROM mysql.user WHERE user='nova';
+------+--------------------+
| User | Host               |
+------+--------------------+
| nova | %                  |
| nova | openstack.test.com |
+------+--------------------+
2 rows in set (0.001 sec)

MariaDB [(none)]> SHOW DATABASES LIKE 'nova%';
+------------------+
| Database (nova%) |
+------------------+
| nova             |
| nova_api         |
| nova_cell0       |
+------------------+
3 rows in set (0.000 sec)
```
- DB 비번 변경
```
ALTER USER 'nova'@'%' IDENTIFIED BY 'c0d23266b662473a';
ALTER USER 'nova'@'openstack.test.com' IDENTIFIED BY 'c0d23266b662473a';
FLUSH PRIVILEGES;
```
- 정상 적용 확인
```
mysql -unova -p'c0d23266b662473a' -h openstack.test.com -e "SELECT 1;"
```
## DB 1045 인증 에러
- 설치하다가 중간 실패하면 생길 수 있음
- /etc/nova/nova.conf 지정된 패스워드
- 아래 명령어로 패스워드 확인
```
# DB 커넥션 라인 확인
grep -nE "^\[database\]|\[api_database\]|connection" /etc/nova/nova.conf | egrep "mysql\+pymysql"
```
- answer file에 있는 패스워드 세 개를 동일하게 설정
- nova 계정 두 개도 비번 동일하게 설정
```
mysql -uroot
ALTER USER 'nova'@'openstack.test.com' IDENTIFIED BY '1a71bf8b2ab14714';
ALTER USER 'nova'@'%' IDENTIFIED BY '1a71bf8b2ab14714';
FLUSH PRIVILEGES;
```
- 설치할 때 answerfile을 고정해서 설치
	- 그렇지 않으면 멋대로 패스워드 생성해서 루프 발생
# 설치 완료
- 확인
```
[root@openstack ~]# packstack --answer-file=/root/packstack-answer.txt
Welcome to the Packstack setup utility

The installation log file is available at: /var/tmp/packstack/20260128-230528-6mwhhkx6/openstack-setup.log

Installing:
Clean Up                                             [ DONE ]
Discovering ip protocol version                      [ DONE ]
Setting up ssh keys                                  [ DONE ]
Preparing servers                                    [ DONE ]
Pre installing Puppet and discovering hosts' details [ DONE ]
Preparing pre-install entries                        [ DONE ]
Installing time synchronization via NTP              [ DONE ]
Setting up CACERT                                    [ DONE ]
Preparing AMQP entries                               [ DONE ]
Preparing MariaDB entries                            [ DONE ]
Fixing Keystone LDAP config parameters to be undef if empty[ DONE ]
Preparing Keystone entries                           [ DONE ]
Preparing Glance entries                             [ DONE ]
Preparing Nova API entries                           [ DONE ]
Creating ssh keys for Nova migration                 [ DONE ]
Gathering ssh host keys for Nova migration           [ DONE ]
Preparing Nova Compute entries                       [ DONE ]
Preparing Nova Scheduler entries                     [ DONE ]
Preparing Nova VNC Proxy entries                     [ DONE ]
Preparing OpenStack Network-related Nova entries     [ DONE ]
Preparing Nova Common entries                        [ DONE ]
Preparing Neutron API entries                        [ DONE ]
Preparing Neutron L3 entries                         [ DONE ]
Preparing Neutron L2 Agent entries                   [ DONE ]
Preparing Neutron DHCP Agent entries                 [ DONE ]
Preparing Neutron Metering Agent entries             [ DONE ]
Checking if NetworkManager is enabled and running    [ DONE ]
Preparing OpenStack Client entries                   [ DONE ]
Preparing Horizon entries                            [ DONE ]
Preparing Gnocchi entries                            [ DONE ]
Preparing Redis entries                              [ DONE ]
Preparing Ceilometer entries                         [ DONE ]
Preparing Aodh entries                               [ DONE ]
Preparing Puppet manifests                           [ DONE ]
Copying Puppet modules and manifests                 [ DONE ]
Applying 192.168.211.106_controller.pp
192.168.211.106_controller.pp:                       [ DONE ]
Applying 192.168.211.106_network.pp
192.168.211.106_network.pp:                          [ DONE ]
Applying 192.168.211.106_compute.pp
192.168.211.106_compute.pp:                          [ DONE ]
Applying 192.168.211.106_controller_post.pp
192.168.211.106_controller_post.pp:                  [ DONE ]
Applying Puppet manifests                            [ DONE ]
Finalizing                                           [ DONE ]

 **** Installation completed successfully ******

Additional information:
 * Parameter CONFIG_NEUTRON_L2_AGENT: You have chosen OVN Neutron backend. Note that this backend does not support the VPNaaS plugin. Geneve will be used as the encapsulation method for tenant networks
 * File /root/keystonerc_admin has been created on OpenStack client host 192.168.211.106. To use the command line tools you need to source the file.
 * To access the OpenStack Dashboard browse to http://192.168.211.106/dashboard .
Please, find your login credentials stored in the keystonerc_admin in your home directory.
 * The installation log file is available at: /var/tmp/packstack/20260128-230528-6mwhhkx6/openstack-setup.log
 * The generated manifests are available at: /var/tmp/packstack/20260128-230528-6mwhhkx6/manifests
```
# 설치 이후 확인
- 환경 변수 추가
	- 인증 정보를 현재 터미널에 적용 (적용 안하면 clil에서 인증 때문에 작업 안됨)
	  keystonerc_admin 파일 내용을 현재 터미널에 적용하는 명령어 
```
 source ~/keystonerc_admin
 
 [root@openstack ~(keystone_admin)]# cat ~/keystonerc_admin
unset OS_SERVICE_TOKEN
    export OS_USERNAME=admin
    export OS_PASSWORD='twoneulbo0510!'
    export OS_REGION_NAME=RegionOne
    export OS_AUTH_URL=http://192.168.211.106:5000/v3
    export PS1='[\u@\h \W(keystone_admin)]\$ '

export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
    [root@openstack ~(keystone_admin)]#
    
    
적용 안하는 경우
[root@openstack ~]# openstack service list
Missing value auth-url required for auth plugin password
```
- 토큰 발급 테스트
	- 인증 테스트나 디버깅 용도로 사용
	- 실제로는 위에 환경변수 추가만 해주면 cli 사용 가능
```
[root@openstack ~(keystone_admin)]# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2026-01-28T15:22:25+0000                                                                                                                                                                |
| id         | gAAAAABpehuhUTKhMamIBhpD2qnK3_a2no3C3NY9SzqfdOSjKe5RTyEBFRwMK9x65pA5X61p4gdzJv55x_3lNxddZT1vVqcB2RlgBj97SmR7pLTZzM5b3IvxshwjOHeA8zazkfai0g7PHPBK7swII-mfjjQq5LsPH5n9a6J70KpRs-36kQfbajM |
| project_id | a309d0dc5ca54274aaa6558e50ac1fbb                                                                                                                                                        |
| user_id    | b0e822be1e364c2dbbc9a3ba86b9f1cc                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
- 서비스 리스트 조회
	- keystone에 등록된 API 서버 목록 (컨트롤 플레인(API 서버)이 살아있는지 조회)
```
[root@openstack ~(keystone_admin)]# openstack service list
+----------------------------------+-----------+--------------+
| ID                               | Name      | Type         |
+----------------------------------+-----------+--------------+
| 529ce3b4a0044c2d9ea41babfda9ef23 | aodh      | alarming     |
| 5f95977bd6e94a0d83e8b1a95d8b428c | keystone  | identity     |
| 6ea4a67d8dfb47e6adfd9e6a86f141ce | placement | placement    |
| 81e8a99e8a3a468a9fcd8910861d1ae2 | cinderv3  | volumev3     |
| 9a16983bed9d4b8fabe2c02461833ca9 | neutron   | network      |
| a482d3e559f34ec2873d638c0b4ae4ba | glance    | image        |
| a5dc6ed888fb4217a03dec50c535293b | gnocchi   | metric       |
| b93ef6ee98a44b63a446b8f364952c4a | nova      | compute      |
| bf21070629624bb4ba61c42dd08547e6 | swift     | object-store |
+----------------------------------+-----------+--------------+
```
- 네트워크 동작 프로세스 정상 동작 확인
```
[root@openstack ~(keystone_admin)]# openstack network agent list
+--------------------------------------+------------------------------+--------------------+-------------------+-------+-------+----------------------------+
| ID                                   | Agent Type                   | Host               | Availability Zone | Alive | State | Binary                     |
+--------------------------------------+------------------------------+--------------------+-------------------+-------+-------+----------------------------+
| c4e92597-eadf-4c0a-bbbd-fa5123dba245 | OVN Controller Gateway agent | openstack.test.com |                   | :-)   | UP    | ovn-controller             |
| 73c34ae1-8a76-5e9a-b44a-9083506bbc03 | OVN Metadata agent           | openstack.test.com |                   | :-)   | UP    | neutron-ovn-metadata-agent |
+--------------------------------------+------------------------------+--------------------+-------------------+-------+-------+----------------------------+
```
- OVN Controller Gateway agent
	- 서비스이름: ovn controller
	- OVN 컨트롤러, 실제 패킷을 포워딩하는 역할
- OVN Metadata agent
	- 서비스이름: neutron-ovn-metadata-agent
	- VM 초기 부팅 시 ssh 키 설정, 패스워드 세팅, 호스트네임, user date, cloud init등 세팅해주는 역할
- compute 서비스 리스트 조회
```
[root@openstack ~(keystone_admin)]# openstack compute service list
+--------------------------------------+----------------+--------------------+----------+---------+-------+----------------------------+
| ID                                   | Binary         | Host               | Zone     | Status  | State | Updated At                 |
+--------------------------------------+----------------+--------------------+----------+---------+-------+----------------------------+
| f3236568-58eb-4783-b885-e6e8d051698e | nova-conductor | openstack.test.com | internal | enabled | up    | 2026-02-10T01:07:31.000000 |
| 4a339dfd-f6b2-444e-9cb6-7f833e9283af | nova-scheduler | openstack.test.com | internal | enabled | up    | 2026-02-10T01:07:35.000000 |
| 696fed2d-464a-42f0-94aa-717c6f940629 | nova-compute   | openstack.test.com | nova     | enabled | down  | 2026-01-26T10:16:07.000000 |
+--------------------------------------+----------------+--------------------+----------+---------+-------+----------------------------+
```
- 볼륨 서비스 리스트 조회
```
[root@openstack ~(keystone_admin)]# openstack volume service list
+------------------+------------------------+------+---------+-------+----------------------------+
| Binary           | Host                   | Zone | Status  | State | Updated At                 |
+------------------+------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | openstack.test.com     | nova | enabled | up    | 2026-02-10T01:07:45.000000 |
| cinder-volume    | openstack.test.com@lvm | nova | enabled | up    | 2026-02-10T01:07:45.000000 |
| cinder-backup    | openstack.test.com     | nova | enabled | up    | 2026-02-10T01:07:52.000000 |
+------------------+------------------------+------+---------+-------+----------------------------+
```
# 대시보드 접속 http 500 에러
- 대시보드 접속 시 500 에러 발생
```
## Something went wrong!

An unexpected error has occurred. Try refreshing the page. If that doesn't help, contact your local administrator.
```
- 원인
	- packstack의 horizon 세팅에서 참조하고 있는 MemcachedCache이 실제 django에서는 현재 사용하지 않음
- 로그 확인
	- tail -f /var/log/horizon/horizon.log
```
2026-02-03 08:34:09,231 1172986 ERROR django.request Internal Server Error: /dashboard/
Traceback (most recent call last):
  File "/usr/lib/python3.9/site-packages/django/utils/module_loading.py", line 30, in import_string
    return cached_import(module_path, class_name)
  File "/usr/lib/python3.9/site-packages/django/utils/module_loading.py", line 16, in cached_import
    return getattr(module, class_name)
AttributeError: module 'django.core.cache.backends.memcached' has no attribute 'MemcachedCache'
```
- 해결
	- /etc/openstack-dashboard/local_settings
```
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
```
# 볼륨(cinder) http 503 에러
- GUI에서 keystone이 사용할 수 없다는 에러가 뜸
```
The server is currently unavailable. Please try again at a later time.<br /><br /> The Keystone service is temporarily unavailable. (HTTP 503)
```
- CLI에서 volume list 조회 실패
	- 확인: 8774(nova) 200, 정상
	  8776(cinder)만 503 응답
```
[root@openstack ~(keystone_admin)]# openstack volume list -vvv

RESP: [200] Connection: Keep-Alive Content-Encoding: gzip Content-Type: application/json Date: Wed, 04 Feb 2026 00:34:51 GMT Keep-Alive: timeout=15, max=100 OpenStack-API-Version: compute 2.1 Server: Apache Transfer-Encoding: chunked Vary: OpenStack-API-Version,X-OpenStack-Nova-API-Version,Accept-Encoding X-OpenStack-Nova-API-Version: 2.1 x-compute-request-id: req-b40e6a00-e82b-4757-9d4d-2f43ea086c5c x-openstack-request-id: req-b40e6a00-e82b-4757-9d4d-2f43ea086c5c RESP BODY: {"servers": []} GET call to compute for http://192.168.211.106:8774/v2.1/servers/detail used request id req-b40e6a00-e82b-4757-9d4d-2f43ea086c5c REQ: curl -g -i -X GET http://192.168.211.106:8776/v3/volumes/detail -H "Accept: application/json" -H "User-Agent: python-cinderclient" -H "X-Auth-Token: {SHA256}87038ff72e0bb323e674f70d89209c4392bffa068f7606c483703b9f9b8b4b9a" Starting new HTTP connection (1): 192.168.211.106:8776 http://192.168.211.106:8776 "GET /v3/volumes/detail HTTP/1.1" 503 218 RESP: [503] Connection: keep-alive Content-Length: 218 Content-Type: application/json Date: Wed, 04 Feb 2026 00:34:52 GMT X-Openstack-Request-Id: req-3e3ea123-18ef-441b-a891-ab6eae59b226 RESP BODY: {"message": "The server is currently unavailable. Please try again at a later time.<br /><br />\nThe Keystone service is temporarily unavailable.\n\n", "code": "503 Service Unavailable", "title": "Service Unavailable"} GET call to volumev3 for http://192.168.211.106:8776/v3/volumes/detail used request id req-3e3ea123-18ef-441b-a891-ab6eae59b226 The server is currently unavailable. Please try again at a later time.<br /><br /> The Keystone service is temporarily unavailable.
```
- 원인
	- keystone에 등록된 cinder 비밀번호와 실제 cinder가 keystone에 접근할 때 사용하는 비밀번호가 달라서 발생
- 확인
	- /etc/cinder/cinder.conf
		- 여기에 cinder가 키스톤에 접근하는 비번, cinder가 DB한테 접근하는 비번 등이 저장되어 있음
```
[keystone_authtoken]
auth_url=http://192.168.211.106:5000
username=cinder
user_domain_name=Default
project_name=services
project_domain_name=Default
password=9220bdf8c16f412f
```
- 위 패스워드로 키스톤 토큰 발급
	- 401 에러 발생
```
openstack token issue \
  --os-username cinder \
  --os-password 9220bdf8c16f412f \
  --os-project-name services \
  --os-auth-url http://192.168.211.106:5000/v3 \
  --os-user-domain-name Default \
  --os-project-domain-name Default
The request you have made requires authentication. (HTTP 401) (Request-ID: req-1b120fcb-efe6-4837-86ac-b063cd6f083d)
```
- 패스워드를 cinder.conf 설정파일에 있는 비번으로 맞춰주기
	- 아래 명령어 쓰면 keystone DB 안의 cinder pw가 변경된다
```
[root@openstack ~(keystone_admin)]# source ~/keystonerc_admin
openstack user set --password 9220bdf8c16f412f cinder

source ~/keystonerc_admin -> admin 계정으로 cinder api 호출할 수 있도록 함
openstack user set --password 9220bdf8c16f412f cinder -> pw 변경하는 명령어
```
- 완료
```
[root@openstack ~(keystone_admin)]# openstack token issue \
  --os-username cinder \
  --os-password 9220bdf8c16f412f \
  --os-project-name services \
  --os-auth-url http://192.168.211.106:5000/v3 \
  --os-user-domain-name Default \
  --os-project-domain-name Default
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2026-02-10T01:45:33+0000                                                                                                                                                                |
| id         | gAAAAABpin-tvj-3ITUHYxtTpnnL02MdXNWeVSdtloecyCNEGMV_VelyPYYpIFIGxqg7v2xbcTiw0UOLJurJD8Y9wDQ5lpGBWY5IqxWTH8a02RZZubrQ_3IypJnsZ1i7jCgix6HRLKpSEv6u2qTKa35f79IFNEn05pMGzn6OqhvqwYUXb6brQXA |
| project_id | e7805ac3c4984cd9bcfae6dcd5514bf0                                                                                                                                                        |
| user_id    | 3c3f4606304948e3853edfe38e607a37                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
[root@openstack ~(keystone_admin)]#
```
# 가상머신 생성
- 이미지, flavor 리스트 조회
```
[root@openstack ~(keystone_admin)]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 04743164-6b1e-417b-85cf-2f24e1721030 | cirros | active |
+--------------------------------------+--------+--------+
[root@openstack ~(keystone_admin)]#
[root@openstack ~(keystone_admin)]# openstack flavor list
+----+-----------+-------+------+-----------+-------+-----------+
| ID | Name      |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+-----------+-------+------+-----------+-------+-----------+
| 1  | m1.tiny   |   512 |    1 |         0 |     1 | True      |
| 2  | m1.small  |  2048 |   20 |         0 |     1 | True      |
| 3  | m1.medium |  4096 |   40 |         0 |     2 | True      |
| 4  | m1.large  |  8192 |   80 |         0 |     4 | True      |
| 5  | m1.xlarge | 16384 |  160 |         0 |     8 | True      |
+----+-----------+-------+------+-----------+-------+-----------+
```
- test vm 생성
	- private 네트워크는 자동으로 생성됨
```
[root@openstack ~(keystone_admin)]# openstack server create \
  --image cirros \
  --flavor m1.tiny \
  --network private \
  test-vm
```
- 생성 확인
```
[root@openstack ~(keystone_admin)]# openstack server list
+--------------------------------------+---------+--------+----------+--------+---------+
| ID                                   | Name    | Status | Networks | Image  | Flavor  |
+--------------------------------------+---------+--------+----------+--------+---------+
| 33d96d33-557f-4ecf-bea6-2f185ad9551e | test-vm | ERROR  |          | cirros | m1.tiny |
+--------------------------------------+---------+--------+----------+--------+---------+
```
## Nova compute 오류 발생 (code 500 error)
- 오류 확인
	- openstack server show test-vm
	- Nova 부분 문제
```
fault                               | {'code': 500, 'created': '2026-02-10T01:23:36Z', 'message': '유효한 호스트가 없습니다. 사용 가능한 호스트가 부족합니다.', 'details': 'Traceback (most recent call last
```
- 로그 확인
	- tail -n 50 /var/log/nova/nova-compute.log
	- 원인: 이전에 이 호스트에서 nova-compute를 가동시킨 이력이 있었으나 지금 로컬 노드 정보 식별 정보가 사라져서 가동이 안 되는 상태
```
2026-02-10 10:34:02.320 3762171 ERROR oslo_service.service nova.exception.InvalidConfiguration: No local node identity found, but this is not our first startup on this host. Refusing to start after potentially having lost that state!
2026-02-10 10:34:02.320 3762171 ERROR oslo_service.service
```
- nova compute 상태 확인
	- nova-compute 다운되어 있었음
```
696fed2d-464a-42f0-94aa-717c6f940629 | nova-compute   | openstack.test.com | nova     | enabled | down  | 2026-01-26T10:16:07.000000
```
- nova-compute 데몬 상태 확인
	- 시작하자마자 크래시 발생해서 계속 다운되는 상태 반복
```
[root@openstack ~(keystone_admin)]# journalctl -u openstack-nova-compute -n 80 --no-pager
 2월 10 10:40:34 openstack.test.com systemd[1]: Stopped OpenStack Nova Compute Server.
 2월 10 10:40:34 openstack.test.com systemd[1]: openstack-nova-compute.service: Consumed 1.867s CPU time, 121.4M memory peak.
 2월 10 10:40:34 openstack.test.com systemd[1]: Starting OpenStack Nova Compute Server...
 2월 10 10:40:36 openstack.test.com systemd[1]: Started OpenStack Nova Compute Server.
 2월 10 10:40:36 openstack.test.com systemd[1]: openstack-nova-compute.service: Deactivated successfully.
 2월 10 10:40:36 openstack.test.com systemd[1]: openstack-nova-compute.service: Consumed 1.882s CPU time, 121.4M memory peak.
 2월 10 10:40:36 openstack.test.com systemd[1]: openstack-nova-compute.service: Scheduled restart job, restart counter is at 431311.
 2월 10 10:40:36 openstack.test.com systemd[1]: Stopped OpenStack Nova Compute Server.
 2월 10 10:40:36 openstack.test.com systemd[1]: openstack-nova-compute.service: Consumed 1.882s CPU time, 121.4M memory peak.
 2월 10 10:40:36 openstack.test.com systemd[1]: Starting OpenStack Nova Compute Server...
```
- nova-compute 데몬 중지 및 실패 로그 초기화
```
systemctl stop openstack-nova-compute
systemctl reset-failed openstack-nova-compute
```
----
### 재부팅으로 인해 트러블 슈팅 다시 시작
- nova-compute 로그 확인 결과 rabbitmq가 down된 상태 확인
```
2026-02-27 16:36:24.619 1431 ERROR oslo.messaging._drivers.impl_rabbit [None req-8e3f50b0-9feb-4c49-be6f-95f76410b26b - - - - - -] Connection failed: [Errno 111] ECONNREFUSED (retrying in 31.0 seconds): ConnectionRefusedError: [Errno 111] ECONNREFUSED
```
- 데몬 상태 조회
```
[root@openstack ~(keystone_admin)]# systemctl status rabbitmq-server
× rabbitmq-server.service - RabbitMQ broker
     Loaded: loaded (/usr/lib/systemd/system/rabbitmq-server.service; enabled; preset: disabled)
    Drop-In: /etc/systemd/system/rabbitmq-server.service.d
             └─90-limits.conf
     Active: failed (Result: exit-code) since Fri 2026-02-27 15:39:53 KST; 1h 5min ago
    Process: 1453 ExecStart=/usr/lib/rabbitmq/bin/rabbitmq-server (code=exited, status=1/FAILURE)
   Main PID: 1453 (code=exited, status=1/FAILURE)
     Status: "Startup in progress"
        CPU: 991ms

 2월 27 15:39:52 openstack.test.com rabbitmq-server[1453]: 2026-02-27 15:39:52.117334+09:00 [error] <0.127.0>   neighbours:
 2월 27 15:39:52 openstack.test.com rabbitmq-server[1453]: 2026-02-27 15:39:52.117334+09:00 [error] <0.127.0>
 2월 27 15:39:52 openstack.test.com rabbitmq-server[1453]: 2026-02-27 15:39:52.131154+09:00 [notice] <0.44.0> Application rabbitmq_prelaunch exited with reason: {{shutdown,{failed_to_start_child,prelaunch,>
 2월 27 15:39:53 openstack.test.com rabbitmq-server[1453]: {"Kernel pid terminated",application_controller,"{application_start_failure,rabbitmq_prelaunch,{{shutdown,{failed_to_start_child,prelaunch,{epmd_e>
 2월 27 15:39:53 openstack.test.com rabbitmq-server[1453]: Kernel pid terminated (application_controller) ({application_start_failure,rabbitmq_prelaunch,{{shutdown,{failed_to_start_child,prelaunch,{epmd_er>
 2월 27 15:39:53 openstack.test.com rabbitmq-server[1453]:
 2월 27 15:39:53 openstack.test.com rabbitmq-server[1453]: Crash dump is being written to: erl_crash.dump...done
 2월 27 15:39:53 openstack.test.com systemd[1]: rabbitmq-server.service: Main process exited, code=exited, status=1/FAILURE
 2월 27 15:39:53 openstack.test.com systemd[1]: rabbitmq-server.service: Failed with result 'exit-code'.
 2월 27 15:39:53 openstack.test.com systemd[1]: Failed to start RabbitMQ broker
```
### 문제 해결
- rabbitmq 데몬 시작 후 다시 compute log 확인
	- nova compute가 자신의 identity 파일을 잃어버려서 생긴 문제
		- DB에 저장된 compute_id가 없어서 DB도 에러 로그 발생
	- nova compute는 시작할 때 자신의 고유 id를 저장해두는데 이게 사라졌음
		- `/var/lib/nova/compute_id`
```
2026-02-27 16:48:56.447 88832 ERROR nova.db.main.api [None req-b6582d21-b4c7-493c-9144-2d383c0d4830 - - - - - -] No DB access allowed in nova-compute: File "/usr/lib/python3.9/site-packages/eventlet/greenthread.py", line 264, in main

No local node identity found, but this is not our first startup on this host.
Refusing to start after potentially having lost that state!
```
- DB에서 기존 로그 찾기
	- DB의 nova 유저 패스워드 찾기
		- `/etc/nova/nova.conf`
```
connection=mysql+pymysql://nova:4516a4dcb3ac4f96@192.168.211.106/nova
```
- DB 접속 후 uuid 확인
```
mysql -u nova -p'패스워드' nova

MariaDB [nova]> SELECT uuid, hypervisor_hostname, deleted FROM compute_nodes;
+--------------------------------------+---------------------+---------+
| uuid                                 | hypervisor_hostname | deleted |
+--------------------------------------+---------------------+---------+
| ad38d267-fcc6-4eda-9186-6d33e99fcb12 | openstack.test.com  |       0 |
+--------------------------------------+---------------------+---------+
```
- /etc/nova/nova.conf/compute-id 에 uuid 복사
- 권한/소유자 변경 후 nova compute 재시작
```
chown nova:nova /var/lib/nova/compute_id 
chmod 600 /var/lib/nova/compute_id
systemctl restart openstack-nova-compute
```
- nova compute 정상 복구 확인
	- openstack compute service list
```
[root@openstack nova(keystone_admin)]# openstack compute service list
+--------------------------------------+----------------+--------------------+----------+---------+-------+----------------------------+
| ID                                   | Binary         | Host               | Zone     | Status  | State | Updated At                 |
+--------------------------------------+----------------+--------------------+----------+---------+-------+----------------------------+
| f3236568-58eb-4783-b885-e6e8d051698e | nova-conductor | openstack.test.com | internal | enabled | up    | 2026-02-27T08:43:15.000000 |
| 4a339dfd-f6b2-444e-9cb6-7f833e9283af | nova-scheduler | openstack.test.com | internal | enabled | up    | 2026-02-27T08:43:15.000000 |
| 696fed2d-464a-42f0-94aa-717c6f940629 | nova-compute   | openstack.test.com | nova     | enabled | up    | 2026-02-27T08:43:14.000000 |
+--------------------------------------+----------------+--------------------+----------+---------+-------+----------------------------+
```
## Cinder backup 서비스 down 오류 발생
- 2월 27일에 compute 데몬 살리고 나서 cinder backup 서비스 down 발생
- 상태 확인
```
[root@openstack cinder(keystone_admin)]# openstack volume service  list
+------------------+------------------------+------+---------+-------+----------------------------+
| Binary           | Host                   | Zone | Status  | State | Updated At                 |
+------------------+------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | openstack.test.com     | nova | enabled | up    | 2026-03-03T00:55:09.000000 |
| cinder-volume    | openstack.test.com@lvm | nova | enabled | up    | 2026-03-03T00:55:11.000000 |
| cinder-backup    | openstack.test.com     | nova | enabled | down  | 2026-02-27T07:24:01.000000 |
+------------------+------------------------+------+---------+-------+----------------------------+
```
- 데몬 상태 (정상)
```
[root@openstack cinder(keystone_admin)]# systemctl status openstack-cinder-backup
● openstack-cinder-backup.service - OpenStack Cinder Backup Server
     Loaded: loaded (/usr/lib/systemd/system/openstack-cinder-backup.service; enabled; preset: disabled)
     Active: active (running) since Fri 2026-02-27 16:23:50 KST; 3 days ago
   Main PID: 57766 (cinder-backup)
      Tasks: 1 (limit: 148067)
     Memory: 99.5M (peak: 99.9M)
        CPU: 1min 12.251s
     CGroup: /system.slice/openstack-cinder-backup.service
             └─57766 /usr/bin/python3 -s /usr/bin/cinder-backup --config-file /usr/share/cinder/cinder-dist.conf --config-file /etc/cinder/cinder.conf --logfile /var/log/cinder/backup.log

Notice: journal has been rotated since unit was started, output may be incomplete.
```
- 로그 확인 (/var/log/cinder/backup.log)
	- less /var/log/cinder/backup.log
```
2026-02-27 15:44:56.415 7201 ERROR cinder oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:44:56.415 7201 ERROR cinder (Background on this error at: https://sqlalche.me/e/14/e3q8)
2026-02-27 15:44:56.415 7201 ERROR cinder
2026-02-27 15:44:57.843 9262 INFO cinder.cmd.backup [-] Backup running in single process mode.
2026-02-27 15:44:57.895 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 10 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:45:07.908 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 9 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:45:17.920 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 8 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:45:27.925 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 7 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:45:37.935 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 6 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:45:47.946 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 5 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:45:57.954 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 4 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:46:07.965 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 3 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:46:17.976 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 2 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:46:27.978 9262 WARNING oslo_db.sqlalchemy.engines [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] SQL connection failed. 1 attempts left.: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
2026-02-27 15:46:37.989 9262 CRITICAL cinder [None req-a32f39a6-e2a9-4175-a969-615af19de307 - - - - - -] Unhandled error: oslo_db.exception.DBConnectionError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server on '192.168.211.106' ([Errno 101] ENETUNREACH)")
(Background on this error at: https://sqlalche.me/e/14/e3q8)
```
- 데몬 재시작 후 해결
	- systemctl restart openstack-cinder-backup

## 이미지 파일 삭제 인해 가상머신 생성 실패
- test-vm 재생성 실패 
	- openstack server show test-vm
```
   fault                               | {'code': 500, 'created': '2026-03-03T00:57:53Z', 'message': '인스턴스 7964ec1e-46f8-40da-b3f8-956e86637e9c의 빌드가 중단됨: 04743164-6b1e-417b-85cf-2f24e1721030 이미지는 허용할 수 없음: |
```
- 이미지 상태는 active
```
[root@openstack cinder(keystone_admin)]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 04743164-6b1e-417b-85cf-2f24e1721030 | cirros | active |
+--------------------------------------+--------+--------+
```
- 실제 이미지 파일 존재x
	- /var/lib/glance/images/
```
[root@openstack images(keystone_admin)]# ls -l /var/lib/glance/images/
합계 0
```
- 기존 이미지 삭제 후 재 다운로드
```
openstack image delete 04743164-6b1e-417b-85cf-2f24e1721030
wget http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img
```
- 기존 vm 삭제
	- openstack server delete test-vm
- 이미지 재생성
	- file에는 실제 이미지 파일 경로 지정
	- container-format bare: 컨테이너로 생성하지 않음
	- --public: 이미지 전체 공개
```
[root@openstack images(keystone_admin)]# openstack image create "cirros" \
  --file cirros-0.5.2-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --public
```
- 확인
```
[root@openstack images(keystone_admin)]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 97e72f79-2c5a-45b4-8678-87f9c5b37053 | cirros | active |
+--------------------------------------+--------+--------+
```
## 가상머신 재생성
### 네트워크 확인
- 생성하기 전에 네트워크 확인
	- public, private 네트워크 기본으로 생성해주며 private 네트워크 대역은 10.0.0.0/24
```
[root@openstack images(keystone_admin)]# openstack network list
+--------------------------------------+---------+--------------------------------------+
| ID                                   | Name    | Subnets                              |
+--------------------------------------+---------+--------------------------------------+
| 12d579c8-b8b1-415b-89a2-3c02b86dfcdd | public  | aea6a17a-594a-4690-b367-ffad26b39c3f |
| cc1631c7-f8eb-4fb6-8a8a-dd26bbbe2c14 | private | 9beebcf0-0d6e-4389-9363-7bddb5174f9c |
+--------------------------------------+---------+--------------------------------------+

[root@openstack images(keystone_admin)]# openstack subnet list
+--------------------------------------+----------------+--------------------------------------+---------------+
| ID                                   | Name           | Network                              | Subnet        |
+--------------------------------------+----------------+--------------------------------------+---------------+
| 9beebcf0-0d6e-4389-9363-7bddb5174f9c | private_subnet | cc1631c7-f8eb-4fb6-8a8a-dd26bbbe2c14 | 10.0.0.0/24   |
| aea6a17a-594a-4690-b367-ffad26b39c3f | public_subnet  | 12d579c8-b8b1-415b-89a2-3c02b86dfcdd | 172.24.4.0/24 |
+--------------------------------------+----------------+--------------------------------------+---------------+
```
- private 네트워크 shared로 설정
	- openstack network set --share private
	- private 네트워크는 오픈스택에서 자동으로 demo 프로젝트에 생성해서
	  admin으로 로그인한 horizon에서는 보이지 않음
### 가상머신 생성 완료
- 가상머신 생성
```
openstack server create \
  --image cirros \
  --flavor m1.tiny \
  --network private \
  test-vm
```
- 상태 확인
	- ip는 10.0.0.0 대역 안에서 dhcp로 할당
```
[root@openstack images(keystone_admin)]# openstack server list
+--------------------------------------+---------+--------+-------------------+--------+---------+
| ID                                   | Name    | Status | Networks          | Image  | Flavor  |
+--------------------------------------+---------+--------+-------------------+--------+---------+
| 7bb7c8c7-3b02-4d15-8b2e-d5ac0208d20f | test-vm | ACTIVE | private=10.0.0.61 | cirros | m1.tiny |
+--------------------------------------+---------+--------+-------------------+--------+---------+
```

# 네트워크 구성 변경으로 인한 로그 폭증
- 원인
	- 네트워크 구조 변경으로 인해 redis 연결 실패
	- aodh(알람)이 계속 재연결 시도
		- /var/log/aodh 로그 폭증
	- / 루트 시스템 100% 발생
- 해결
	- 디스크(qcow2) +50G 확장
	- 파티션 확장
	- pvresize 시도 -> 로그 폭증으로 인해 실패
		- aodh 서비스 중지
		- /var/log/aodh 로그 삭제 후 성공

## 해결 과정 정리
- qcow2 디스크 확장
	- qemu-img resize /home/packstack.qcow2 +50G
- 파티션/파일 시스템 확장 필요
	- lsblk, df-h로 디스크 공간 확인
		- vda는 늘어났지만 vda2에는 적용 안 됨
	- 파티션 확장
```
파티션 확장
parted /dev/vda
resizepart 2 100%
partprobe

LVM 확장
pvresize /dev/vda2
lvextend -l +100%FREE /dev/mapper/rl-root
xfs_growfs /

print로 확인
```
- 이 때 이미 루트가 너무 많이 차서 확장 실패
```
1.원인 확인
du -xhd1 / | sort -h

52G  /var
60G  /

2. /var에서 가장 많이 차지하는 부분 확인
du -xhd1 /var | sort -h

-> /var/log 원인 확정
```
- aodh 서비스 종료
```
systemctl stop openstack-aodh-api
systemctl stop openstack-aodh-evaluator
systemctl stop openstack-aodh-notifier
```
- 로그 폭증 원인 확인
```
redis.exceptions.ConnectionError:
Error 101 connecting to 192.168.211.106:6379
Network is unreachable
```
- 로그 삭제
# 설치 후 주요 데몬 확인
```
MariaDB (데이터베이스): systemctl status mariadb
RabbitMQ (메시지 브로커): systemctl status rabbitmq-server
Memcached (토큰 캐싱): systemctl status memcached
Apache (Keystone 실행):systemctl status httpd
Neutron 서버: systemctl status neutron-server
OVN 북쪽 DB: systemctl status ovn-northd
OVN 컨트롤러 (각 노드 실행): systemctl status ovn-controller
```
- 엔드포인트 리스트
	- openstack endpoint list