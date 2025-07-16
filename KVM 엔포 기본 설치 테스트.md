## 단계
- [x] 패키지 설치, 버전 확인
	- [x] mysql 설치
- [x] Apache
- [ ] DNS(named)
- [ ] sendmail
- [ ] dovecot
## VM 정보
- rocky9.6
	- rocky10은 kvm에서 인식 안 됨
	- 192.168.193.102/24
- 아래와 같이 설치
```
virt-install --name enpotest --memory 8012 --vcpu 2 --cdrom /Rocky-9.6-x86_64-dvd.iso --network bridge=br0 --os-variant rocky9 --disk /home/enpotest,size=50
```
## 패키지 설치
- dnf -y update
- dnf -y install 패키지명
	- dovecot, postfix, httpd, 등
- systemctl enable `데몬명`
```
[root@localhost ~]# systemctl enable named
```
### mysql 설치
- mysqyl 공식 리포지토리 패키지 다운 필요
	- repo 경로: `/etc/yum.repos.d/mysql-community.repo`
```
dnf install -y https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm sudo //mysql repo 파일 설치 패키지
dnf install -y mysql-community-server //repo파일에 들어있는 url을 통해 mysql 최신 버전 다운로드
```
- 오류
	- mysql 다운로드 후 gpg key 관련 에러 발생
```
GPG key at file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql-2022 (0x3A79BD29) is already installed
The GPG keys listed for the "MySQL 8.0 Community Server" repository are already installed but they are not correct for this package.

Public key for mysql-community-server-8.0.42-1.el9.x86_64.rpm is not installed. Failing package is: mysql-community-server-8.0.42-1.el9.x86_64
 GPG Keys are configured as: file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql-2022
The downloaded packages were saved in cache until the next successful transaction.
You can remove cached packages by executing 'dnf clean packages'.
Error: GPG check FAILED
```
- 원인
	- 공식 repo rpm의 gpg key 인증서 기간이 만료 됨
```
[root@localhost etc]# gpg --quiet --with-fingerprint /etc/pki/rpm-gpg/RPM-GPG-KEY-mysql-2022
pub   rsa4096 2021-12-14 [SC] [expired: 2023-12-14]
uid           MySQL Release Engineering <mysql-build@oss.oracle.com>
sub   rsa4096 2021-12-14 [E] [expired: 2023-12-14]
```
- 해결
	- gpg key 2023 버전으로 갱신 (수동 등록)
		- 인증서를 내부적으로 rpm db에만 저장
```
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
```
## 버전 확인
```
[root@localhost ~]# httpd -v
Server version: Apache/2.4.62 (Rocky Linux)
Server built:   Jan 29 2025 00:00:00

[root@localhost ~]# mysqld --version
mysqld  Ver 10.5.27-MariaDB for Linux on x86_64 (MariaDB Server)

[root@localhost ~]# dovecot --version
2.3.16 (7e2e900c1a)

[root@localhost etc]# sendmail -d0.1 -bv root
Version 8.16.1

[root@localhost ~]# named -v
BIND 9.16.23-RH (Extended Support Version) <id:fde3b1f>

[root@localhost etc]# mysql --version
mysql  Ver 8.0.42 for Linux on x86_64 (MySQL Community Server - GPL)
```
### 버전 업그레이드 (필요X)
- mariadb /v10.11.13 버전 설치 방법
	- 서비스 중지 후 기존 패키지 삭제
```
[root@localhost ~]# systemctl stop mariadb
[root@localhost ~]# dnf remove mariadb-server mariadb
```
- mariadb 공식 저장소 등록
```
[root@localhost ~]# sudo tee /etc/yum.repos.d/MariaDB.repo <<EOF
> [mariadb]
> name = MariaDB
> baseurl = http://yum.mariadb.org/10.11/rhel9-amd64
> gpgkey = https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
> gpgcheck = 1
> enabled=1
> EOF
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.11/rhel9-amd64
gpgkey = https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck = 1
enabled=1
```
- 재설치
```
dnf install -y MariaDB-server MariaDB-client

[root@localhost ~]# mariadb --version
mariadb  Ver 15.1 Distrib 10.11.13-MariaDB, for Linux (x86_64) using  EditLine wrapper

[root@localhost ~]# systemctl enable mariadb
```

## 방화벽 설정
- 수동 설정 방법
	- 참고: 서비스명은 포트 넘버와 동일 (내부적으로 서비스 정의가 되어 있음)
```
[root@localhost etc]# firewall-cmd --permanent --add-service=http
[root@localhost etc]# firewall-cmd --permanent --add-port=3306/tcp
```
- zone 파일에 직접 추가
	- `/etc/firewalld/zones/public.xml`
```
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
  <service name="cockpit"/>
  <service name="http"/>
  <service name="https"/>
  <port port="3306" protocol="tcp"/>
  <port protocol="tcp" port="25"/>
  <port protocol="tcp" port="110"/>
  <port protocol="tcp" port="143"/>
  <port protocol="tcp" port="993"/>
  <port protocol="tcp" port="995"/>
  <port protocol="tcp" port="53"/>
  <port protocol="udp" port="53"/>
  <port protocol="udp" port="123"/>
  <forward/>
</zone>
```
- 적용, 확인
```
[root@localhost zones]# systemctl restart firewalld
[root@localhost zones]#
[root@localhost zones]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp1s0
  sources:
  services: cockpit dhcpv6-client http https ssh
  ports: 3306/tcp 25/tcp 110/tcp 143/tcp 993/tcp 995/tcp 53/tcp 53/udp 123/udp
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
## Apache 구성
### 기본 경로 확인
- `/var/www/html` :웹 페이지
- `/etc/httpd/conf/httpd.conf` : 설정 파일
	- 기본 문서 루트 설정 확인
```
[root@localhost html]# grep '^DocumentRoot' /etc/httpd/conf/httpd.conf
DocumentRoot "/var/www/html"
```
### 테스트 페이지 생성 
- 위의 경로에 index.html 파일 생성
```
[root@localhost html]# cat index.html
<h1> apache test page </h1>
```
### 시스템 시작
```
[root@localhost html]# systemctl start httpd
[root@localhost html]#
[root@localhost html]#
[root@localhost html]# systemctl status httpd
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Wed 2025-07-16 02:51:57 EDT; 10s ago
       Docs: man:httpd.service(8)
   Main PID: 88437 (httpd)
     Status: "Total requests: 0; Idle/Busy workers 100/0;Requests/sec: 0; Bytes served/sec:   0 B/sec"
      Tasks: 177 (limit: 47605)
     Memory: 25.3M
        CPU: 103ms
     CGroup: /system.slice/httpd.service
             ├─88437 /usr/sbin/httpd -DFOREGROUND
             ├─88438 /usr/sbin/httpd -DFOREGROUND
             ├─88439 /usr/sbin/httpd -DFOREGROUND
             ├─88440 /usr/sbin/httpd -DFOREGROUND
             └─88441 /usr/sbin/httpd -DFOREGROUND

Jul 16 02:51:57 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
Jul 16 02:51:57 localhost.localdomain httpd[88437]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using localhost.localdomain. Set the 'ServerName' directive global>
Jul 16 02:51:57 localhost.localdomain httpd[88437]: Server configured, listening on: port 80
Jul 16 02:51:57 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
```
## DNS 구성
### 기본 구성 파일 위치
- `/etc/named.conf`: 메인 설정 파일
	- 포트 수신 등 옵션 설정
	- 실제 zone 파일을 include하여 zone 정보 구성
- `/etc/named.rfc1912.zones` : 각 도메인이 어떤 파일을 읽을지 정의
- `/var/named/xxx.zone`: 실제 도메인 정보(A,MX)를 담고 있는 zone 파일
	- 생성할 도메인에 대하여 직접 zone 파일 작성 필요
- `/etc/resolv.conf` : DNS 클라이언트용 설정 파일 / 네임서버 정의 (내가 dns 질의 할 서버 지정)
	- 테스트 할 땐 내 nameserver 넣어주기
- 역방향, 정방향
	- 정방향(forward DNS): 도메인 -> ip 주소 변환
	- 역방향(reverse DNS): IP주소-> 도메인 변환
	- 실제 둘 다 지정해주어야 함
### 동작 방식
- named.conf 파일을 읽고 전역 설정 적용
- include로 rfc1912.zones 포함
- 1912.zones 파일에서 각 도메인에 대한 zone 파일로 연결
- /var/named에 있는 실제 zone 데이터 로드
- 사용자 요청 -> bind가 zone 파일 참고하여 응답
### 설정 파일 수정
- `/etc/named.conf`
	- 아래의 localhost 주소를 `any` 로 변경
	- 모두가 해당 도메인을 질의할 수 있도록 함
```
options { listen-on port 53 { 127.0.0.1; }; 
listen-on-v6 port 53 { ::1; }; 
directory "/var/named"; dump-file "/var/named/data/cache_dump.db"; 
statistics-file "/var/named/data/named_stats.txt"; 
memstatistics-file "/var/named/data/named_mem_stats.txt"; 
secroots-file "/var/named/data/named.secroots"; 
recursing-file "/var/named/data/named.recursing"; 
allow-query { localhost; };
```
- `/etc/named.rfc1912.zones`
	- zone 선언이란?
		- 내 DNS 서버가 특정 도메인에 대한 권한을 가지는 것
	- test 도메인 생성 (zone 선언)
		- 하단에 아래와 같이 추가 (정방향, 역방향)
			- 만일 호스트와 네임서버 **ip 대역이 다르면 역방향은 존파일 따로 생성**
```
zone "test" IN{
        type master;
        file "test.zone";
        allow-update { none; };
};

zone "193.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.193.rev.zone";
    allow-update { none; };
};
```
- `/var/named/test.zone` (정방향)
	- 아래와 같이 파일 생성
		- 실제로 네임서버를 정의한 파일, 질의가 들어오면 이 파일을 보고 어느 ip로 매핑 시킬지 결정
```
$TTL 86400
@   IN  SOA     ns1.test. admin.test. (
                2025071601 ; Serial
                3600       ; Refresh
                1800       ; Retry
                604800     ; Expire
                86400 )    ; Minimum

    IN  NS      ns1.test. //test.com 도메인의 네임서버 지정
    IN  A       192.168.193.102 //test.com으로도 접속 가능하도록 함
    IN  MX  10  mail.test. //메일서버 지정
ns1 IN  A       192.168.193.102 //ns1.test.com이 가지는 ip
www IN  A       192.168.193.102 //www 호스트의 ip
mail IN A       192.168.193.102
```
- `/var/named/192.168.193.rev.zone` (역방향)
	- 역방향은 ip에 대해서 하나의 도메인만 지정 가능
```
$TTL 86400
@   IN  SOA ns1.test.com. admin.test.com. (
        2025071601 ; Serial
        3600
        1800
        604800
        86400 )

    IN  NS  ns1.test.com.
102 IN  PTR mail.test.com.
```
- `/etc/resolv.conf`
	- nameserver 주소 넣어주기
- 

