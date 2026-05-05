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
## 네트워크 매니저 disable
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
## Nova compute 오류 발생
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
