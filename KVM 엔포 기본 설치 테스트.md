## 단계
- [x] 패키지 설치, 버전 확인
	- [x] mysql 설치
- [x] Apache
- [x] DNS(named)
- [ ] sendmail
- [ ] dovecot
## VM 정보
- **rocky9.6**
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
### mysql 설치 (8.0.42)
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
### mysql 설치 (5.7)
- 8.x 번대 패키지 제거
```
systemctl stop mysqld

[root@localhost ~]# dnf list installed | grep mysql //설치 된 msyql 패키지 확인
mysql-community-client.x86_64                    8.0.42-1.el9                        @mysql80-community
mysql-community-client-plugins.x86_64            8.0.42-1.el9                        @mysql80-community
mysql-community-common.x86_64                    8.0.42-1.el9                        @mysql80-community
mysql-community-icu-data-files.x86_64            8.0.42-1.el9                        @mysql80-community
mysql-community-libs.x86_64                      8.0.42-1.el9                        @mysql80-community
mysql-community-server.x86_64                    8.0.42-1.el9                        @mysql80-community
mysql80-community-release.noarch                 el9-1                               @@commandline

[root@localhost ~]# dnf remove mysql-community* mysql80*
```
- 5.7 레포지토리(저장소) 설치
	- 로키9에서는 5.7 공식 저장소 없음, 로키8버전에서는 mysql5.7 제거
	- 아래 저장소로 설치 (7.11)
```
wget http://repo.mysql.com/mysql57-community-release-el7-11.noarch.rpm
dnf -y install mysql57-community-release-el7-11.noarch.rpm
```
- 패키지 설치
	- mysql57-community enable 후 설치
	- `dnf install mysql-community-server`
```
[root@localhost ~]# dnf config-manager --disable mysql80-community
[root@localhost ~]# dnf config-manager --enable mysql57-community


//저장소 확인
[root@localhost ~]# dnf repolist --all | grep mysql
mysql-cluster-7.5-community        MySQL Cluster 7.5 Community          disabled
mysql-cluster-7.5-community-source MySQL Cluster 7.5 Community - Source disabled
mysql-cluster-7.6-community        MySQL Cluster 7.6 Community          disabled
mysql-cluster-7.6-community-source MySQL Cluster 7.6 Community - Source disabled
mysql-connectors-community         MySQL Connectors Community           enabled
mysql-connectors-community-source  MySQL Connectors Community - Source  disabled
mysql-tools-community              MySQL Tools Community                enabled
mysql-tools-community-source       MySQL Tools Community - Source       disabled
mysql-tools-preview                MySQL Tools Preview                  disabled
mysql-tools-preview-source         MySQL Tools Preview - Source         disabled
mysql55-community                  MySQL 5.5 Community Server           disabled
mysql55-community-source           MySQL 5.5 Community Server - Source  disabled
mysql56-community                  MySQL 5.6 Community Server           disabled
mysql56-community-source           MySQL 5.6 Community Server - Source  disabled
mysql57-community                  MySQL 5.7 Community Server           enabled    --->확인
mysql57-community-source           MySQL 5.7 Community Server - Source  disabled
mysql80-community                  MySQL 8.0 Community Server           disabled
mysql80-community-source           MySQL 8.0 Community Server - Source  disabled

//속도 느릴 경우
dnf config-manager --save --setopt=timeout=1000
```
#### 호환성 문제
- signal 11 에러 발생
	- 버전 호환성 때문에 발생 (사용하는 라이브러리 다름)
	- 아래 패키지 설치 시 임시 조치 가능
		- 로키8 설치하는 것이 안정
```
//에러 확인
[root@localhost ~]# cat /var/log/mysqld.log | tail -50
2025-08-04T05:55:29.440092Z 0 [Note] InnoDB: Number of pools: 1
2025-08-04T05:55:29.440171Z 0 [Note] InnoDB: Using CPU crc32 instructions
2025-08-04T05:55:29.441334Z 0 [Note] InnoDB: Initializing buffer pool, total size = 128M, instances = 1, chunk size = 128M
2025-08-04T05:55:29.446539Z 0 [Note] InnoDB: Completed initialization of buffer pool
2025-08-04T05:55:29.448129Z 0 [Note] InnoDB: If the mysqld execution user is authorized, page cleaner thread priority can be changed. See the man page of setpriority().
05:55:29 UTC - mysqld got signal 11 ;
This could be because you hit a bug. It is also possible that this binary

//패키지 설치
dnf install -y libaio ncurses-compat-libs compat-openssl11

//데이터 디렉토리 초기화 후 데몬 재시작
rm -rf /var/lib/mysql/*
mysqld --initialize --user=mysql
chown -R mysql:mysql /var/lib/mysql

systemctl restart mysqld
```

## 버전 확인
```
[root@localhost ~]# httpd -v
Server version: Apache/2.4.62 (Rocky Linux)
Server built:   Jan 29 2025 00:00:00

[root@localhost ~]# mysqld --version
mysqld  Ver 5.7.44 for Linux on x86_64 (MySQL Community Server (GPL))

[root@localhost ~]# dovecot --version
2.3.16 (7e2e900c1a)

[root@localhost etc]# sendmail -d0.1 -bv root
Version 8.16.1

[root@localhost ~]# named -v
BIND 9.16.23-RH (Extended Support Version) <id:fde3b1f>

[root@localhost ~]# mysql --version
mysql  Ver 14.14 Distrib 5.7.44, for Linux (x86_64) using  EditLine wrapper
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
- 도메인은 역순으로 읽음 (최상위 도메인이 가장 나중에 나옴)
	- ex) www.test.com. 
		- 루트도메인(보통 생략) `.`> `com`>`test`>`www`
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
zone "test.com" IN{
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
	- zone파일에서 "test.com"으로 정의했기 때문에 ns1, www, mail 뒤에 test.com이 자동으로 붙는 구조
	- MX(메일서버)는 반드시 우선순위 지정 필요
```
$TTL 86400
@   IN  SOA     ns1.test. admin.test. (
                2025071601 ; Serial
                3600       ; Refresh
                1800       ; Retry
                604800     ; Expire
                86400 )    ; Minimum

    IN  NS      ns1.test.com //test.com 도메인의 네임서버 지정
    IN  A       192.168.193.102 //test.com으로도 접속 가능하도록 함
    IN  MX  10  mail.test.com //메일서버 지정
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
	- nameserver 주소 지정
### 확인
- DNS 구문 확인 명령어
	- `named-checkzone test.com /var/named/test.zone`
- nslookup 결과
```
[root@localhost ~]# nslookup www.test.com 192.168.193.102
Server:         192.168.193.102
Address:        192.168.193.102#53

Name:   www.test.com
Address: 192.168.193.102

[root@localhost ~]# nslookup mail.test.com 192.168.193.102
Server:         192.168.193.102
Address:        192.168.193.102#53

Name:   mail.test.com
Address: 192.168.193.102
```
## Sendmail 구성
- 테스트 유저 생성 / pw:1234
```
useradd test

[root@localhost mail]# passwd --stdin test
Changing password for user test.
1234
passwd: all authentication tokens updated successfully.
```
- 메일 송수신 테스트
	- s-nail 패키지 설치 필요
```
//test라는 메일 제목으로 test 유저에게 testmail 메일 전송
echo "testmail" | mail -s "test" test

인터렉티브로 보낼 땐 내용 입력하고 . <- 종료
```
- test 계정 메일 기본 경로 수정
	- /var/mail/root로 되어있는 경우 있음
```
[root@localhost mail]# su test
[test@localhost mail]$ echo 'export MAIL=/var/spool/mail/test' >> ~/.bash_profile
[test@localhost mail]$ source ~/.bash_profile
[test@localhost mail]$ echo $MAIL
/var/spool/mail/test
```
- 확인
```
[test@localhost mail]$ mail
s-nail version v14.9.22.  Type `?' for help
/var/spool/mail/test: 1 message
▸   1 root                  2025-08-04 03:48   19/746   "test     
```
## DB(mysql)
### 초기 설정
- mysql5.7은 설치 시 임시 비밀번호를 자동 생성
```
sudo grep 'temporary password' /var/log/mysqld.log
2025-08-04T06:57:32.973961Z 1 [Note] A temporary password is generated for root@localhost: Hj9YKsgieU%n
```
- `mysql_secure_installation` 로 비밀번호 설정 
	- 임시 비번:Admin123!@#

------------------------
# CentOS 7.9 minimal 설치 방법 정리
## 버전
- php v5.6
- mysql v5.7
- openssl v1.0.2k-fips
## OS 설치 및 저장소 추가
- 192.168.211.105 / root, Admin123!@#
- KVM으로 설치
- 기본 저장소 추가 필요
## php v5.6 설치 방법
- 저장소
	- webatic 저장소
	- webatic 릴리즈 레포 추가 후 yum으로 php 설치
```
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
yum install php56w php56w-cli php56w-common php56w-mbstring php56w-mysql php56w-gd
```
## mysql v5.7 설치 방법
- mysql 저장소 설치
```
wget http://repo.mysql.com/mysql57-community-release-el7-11.noarch.rpm
yum -y install mysql57-community-release-el7-11.noarch.rpm
```
- 버전 확인
```
[root@localhost ~]# php -v
PHP 5.6.40 (cli) (built: Jan 12 2019 13:11:15)
Copyright (c) 1997-2016 The PHP Group
Zend Engine v2.6.0, Copyright (c) 1998-2016 Zend Technologies
```
- mysql 서버 설치
	- gpg key 문제로 인해 수동으로 mysql repo에 2022년도 gpg key 추가 필요
	- centos7은 2023년도 key 인식 불가
```
# mysql 서버 설치
yum install -y yum-utils
yum config-manager --disable mysql80-community
yum config-manager --enable mysql57-community
yum -y install mysql-community-server
--> 인증서 문제 해결 후 설치

#2022년도 인증서 설치
[root@localhost ~]# curl -fsSL https://repo.mysql.com/RPM-GPG-KEY-mysql-2022 -o /etc/pki/rpm-gpg/RPM-GPG-KEY-mysql-2022
[root@localhost ~]# rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-mysql-2022

[root@localhost ~]# vi /etc/yum.repos.d/mysql-community.repo

[mysql57-community]
name=MySQL 5.7 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/7/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql-2022  <---- 해당 부분 파일 경로를 새로 설치한 gpg key 경로로 변경
```
- 설치 완료 / 버전 확인
```
[root@localhost ~]# mysqld --version
mysqld  Ver 5.7.44 for Linux on x86_64 (MySQL Community Server (GPL))
```
------------------

# Rocky8.4 php5.6 설치 방법
- 참고 링크: https://xinet.kr/?p=3924
```
[root@localhost ~]# cat /etc/redhat-release
Rocky Linux release 8.10 (Green Obsidian)

[root@localhost ~]# uname -r
4.18.0-305.3.1.el8_4.x86_64

yum install http://rpms.remirepo.net/enterprise/remi-release-8.rpm

dnf -y install php56-runtime php56-php-fpm php56-php-process php56-php-gd php56-php-common \php56-php-gmp php56-php-pecl-zip php56-php-pear php56-php-mysqlnd  \php56-php-mbstring php56-php-cli php56-php-mcrypt php56-php-pecl-jsonc \php56-php php56-php-dba php56-php-xml php56-php-pdo
```
- 확인
```
[root@localhost ~]# /opt/remi/php56/root/usr/bin/php -v
PHP 5.6.40 (cli) (built: Dec  6 2024 10:27:24)
Copyright (c) 1997-2016 The PHP Group
Zend Engine v2.6.0, Copyright (c) 1998-2016 Zend Technologies

[root@localhost ~]# /opt/remi/php56/root/usr/bin/php -m
[PHP Modules]
bz2
calendar
Core
ctype
curl
date
ereg
exif
fileinfo
filter
ftp
gettext
gmp
hash
iconv
json
libxml
mbstring
mhash
openssl
pcntl
pcre
Phar
readline
Reflection
session
sockets
SPL
standard
tokenizer
zip
zlib

[Zend Modules]
```

# Rockylinux 버전 업그레이드(php,openssl 버전 유지)
- php v5.6
- openssl 1.1.x 유지
- 방법: dnf versionlock 사용

# CentOS 6.9 minimal 설치
- vm 생성
	- root / Admin123!@#
	- 192.168.211.103/24
```
[root@localhost home]# virt-install --name Centos6.9minimal --memory 8012 --vcpu 2 --location /home/CentOS-6.9-x86_64-minimal.iso --graphics vnc --network bridge=br0 --os-variant centos6.9 --disk /home/disk/,size=100

[root@localhost home]# ls -l
total 76365648
-rw-r--r--.  1 qemu qemu    427819008 Mar 28  2017 CentOS-6.9-x86_64-minimal.iso
-rw-r--r--.  1 qemu qemu   1020264448 Jun 10 04:42 CentOS-7-x86_64-Minimal-2009.iso
-rw-------.  1 qemu qemu 107390828544 Sep 16 03:37 disk   --------> centos6.9 disk
-rw-r--r--.  1 root root  53695545344 Sep 10 21:39 enpotest
-rw-r--r--.  1 root root         6852 Jul 18 08:05 enpotest.xml
-rw-------.  1 qemu qemu  64434601984 Sep 16 03:35 rocky8.4disk
-rw-r--r--.  1 qemu qemu   1980760064 Jun 20  2021 Rocky-8.4-x86_64-minimal.iso
-rw-r--r--.  1 qemu qemu  12851544064 May 29 20:06 Rocky-9.6-x86_64-dvd.iso
drwx------. 14 test test         4096 Jul 21 13:13 test
```
## 초기 설정
- ip 할당
	- ip addr add 192.168.211.103/24 dev eth0
	- ip link set eth0 up //기본 상태 인터페이스 down
	- ip route add default 192.168.211.1
- `/etc/sysconfig/network-scripts`
- ssh 접속 / 구형 ssh 모듈을 사용 중이라 현재 클라이언트에서는 해당 모듈을 사용 안 함
	- 신규 서버에서 ssh 접속하지 말고 그냥 세션 열어서 들어가면 된다
- DNS 설정
	- `/etc/resolv.conf`
```
[root@localhost ~]# ssh root@192.168.211.103
Unable to negotiate with 192.168.211.103 port 22: no matching host key type found. Their offer: ssh-rsa,ssh-dss

[root@localhost network-scripts]# cat /etc/resolv.conf
nameserver      8.8.8.8
```
- ip/라우팅 설정 영구 저장(`/etc/sysconfig/network-scripts`)
	- 네트워크매니저 사용 안해서 network-scripts에서 파일 수정 필요
	- 라우팅은 route-eth0 파일 새로 작성
	- ONBOOT=yes : 부팅 시 자동으로 인터페이스를 up
```
[root@localhost /]# cd /etc/sysconfig/network-scripts/
[root@localhost network-scripts]# ls
ifcfg-eth0  ifdown-bnep  ifdown-ipv6  ifdown-ppp     ifdown-tunnel  ifup-bnep  ifup-ipv6  ifup-plusb  ifup-routes  ifup-wireless     network-functions
ifcfg-lo    ifdown-eth   ifdown-isdn  ifdown-routes  ifup           ifup-eth   ifup-isdn  ifup-post   ifup-sit     init.ipv6-global  network-functions-ipv6
ifdown      ifdown-ippp  ifdown-post  ifdown-sit     ifup-aliases   ifup-ippp  ifup-plip  ifup-ppp    ifup-tunnel  net.hotplug


[root@localhost network-scripts]# cat ifcfg-eth0
DEVICE=eth0
HWADDR=52:54:00:7A:6F:09
TYPE=Ethernet
IPADDR=192.168.211.103
NETMASK=255.255.255.0
DNS1=8.8.8.8
UUID=48a45ec3-f276-4023-a723-99737559cbd4
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=static ---> static/none으로 두어야 ip 수동 적용

/etc/sysconfig/network-scripts/route-eth0
[root@localhost network-scripts]# cat route-eth0
0.0.0.0/0 via 192.168.211.1 dev eth0


service network restart
```
## 저장소 추가
- /etc/yum.repos.d
	- https://93it-serverengineer.co.kr/42
	- 두 번째꺼 복사해서 Base.repo 덮어 씌우기
```
[root@localhost yum.repos.d]# cp CentOS-Base.repo CentOS-Base.repo.bk
[root@localhost yum.repos.d]# ls
CentOS-Base.repo  CentOS-Base.repo.bk  CentOS-Debuginfo.repo  CentOS-fasttrack.repo  CentOS-Media.repo  CentOS-Vault.repo
```
## php v5.33 설치
```
yum install -y php php-cli php-common php-mysql php-gd php-xml

[root@localhost network-scripts]# php -v
PHP 5.3.3 (cli) (built: Nov  1 2019 12:28:08)
Copyright (c) 1997-2010 The PHP Group
Zend Engine v2.3.0, Copyright (c) 1998-2010 Zend Technologies
```
## mysql v5.6 설치
- centos6.9 기본 설치: v5.1
- 아래와 같이 v5.1과 php가 의존성 걸려있어서 php-mysqlnd로 설치해야 함
	- v5.6의 특정 라이브러리가 phpv5.33과 버전 안맞음
	- php-mysqlnd는 php에서 제공하는 모듈로 해당 라이브러리 제약을 받지 않음 
```
[root@localhost network-scripts]# yum list installed | grep mysql
mysql-libs.x86_64   5.1.73-8.el6_8      @anaconda-CentOS-201703281317.x86_64/6.9

[root@localhost /]# yum remove mysql-libs-5.1.73-8.el6_8.x86_64
 Package                                   Arch                                  Version                                          Repository                                                               Size
================================================================================================================================================================================================================
Removing:
 mysql-libs                                x86_64                                5.1.73-8.el6_8                                   @anaconda-CentOS-201703281317.x86_64/6.9                                4.0 M
Removing for dependencies:
 php-mysql                                 x86_64                                5.3.3-50.el6_10                                  @updates                                                                216 k
 postfix                                   x86_64                                2:2.6.6-8.el6                                    @anaconda-CentOS-201703281317.x86_64/6.9                                9.7 M

Transaction Summary
================================================================================================================================================================================================================
Remove        3 Package(s)

```
- php-mysqlnd 설치는 실패
	- php-mysqlnd 제공하는 remi repo 사이트 아예 종료됨
```
[root@localhost /]# curl -I https://rpms.remirepo.net/enterprise/6/remi/x86_64/
HTTP/1.1 404 Not Found
Date: Wed, 17 Sep 2025 01:58:35 GMT
Server: Apache/2.4.62 (Rocky Linux) OpenSSL/3.2.2
Content-Type: text/html; charset=iso-8859-1
```
- mysql v5.6 설치
	- wget + rpm -ivh 와 yum install 차이: yum은 의존성 검사해서 패키지 자동으로 설치해준다
	  wget은 rpm을 가져와 로컬에 설치 후 직접 rpm을 설치하는 방식, 만일 의존성 패키지가 필요한 경우 실패 할 수 있음
```
yum -y install https://dev.mysql.com/get/mysql-community-release-el6-5.noarch.rpm

yum install mysql mysql-server mysql-devel -y

mysql-server: mysql 데몬
mysql-devel: 개발용 라이브러리 (php와 연동 위함)
mysql: 클라이언트 툴 (cli 명령어)
```
- 요약
	- 현재 phpv5.33+mysqlv5.6 설치
	- mysql-community-libs 해당 패키지는 설치하면 php5.33과 충돌 가능성 있음
		- mysql56 패키지에서 구버전도 지원하도록 패키지 같이 지원해주어서 문제 없을 거 같다
```
[root@localhost /]# yum list installed  | grep mysql
mysql-community-client.x86_64
                       5.6.51-2.el6     @mysql56-community
mysql-community-common.x86_64
                       5.6.51-2.el6     @mysql56-community
mysql-community-devel.x86_64
                       5.6.51-2.el6     @mysql56-community
mysql-community-libs.x86_64   -----> 신규버전, 이것만 있으면 에러 발생 가능성 있음
                       5.6.51-2.el6     @mysql56-community
mysql-community-libs-compat.x86_64        -------> 구버전 호환
                       5.6.51-2.el6     @mysql56-community
mysql-community-release.noarch
                       el6-5            @/mysql-community-release-el6-5.noarch
mysql-community-server.x86_64
                       5.6.51-2.el6     @mysql56-community
php-mysql.x86_64       5.3.3-50.el6_10  @updates
```

## 버전 정리
- openssl (기본 설치)
	- 1.0.1e-fips
- httpd (기본 설치)
	- 2.2.15
- bind, bind-utils
- sendmail, sendmail-cf
	- v8.14.4
- php v5.33
- mysql v5.6
- dovecot v2.0.9
## 직렬 콘솔 연결
- centos6.9는 grub legacy 사용
- 아래와 같이 변경
	- `/etc/grub.conf`
		- 수정 시 /proc/cmdline에 자동으로 추가
```
/etc/grub.conf

serial --unit=0 --speed=115200
terminal --timeout=5 serial console    --->2줄 추가

default=0
timeout=5
splashimage=(hd0,0)/grub/splash.xpm.gz
hiddenmenu
title CentOS (2.6.32-754.35.1.el6.x86_64)
        root (hd0,0)
        kernel /vmlinuz-2.6.32-754.35.1.el6.x86_64 ro root=/dev/mapper/VolGroup-lv_root rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD rd_LVM_LV=VolGroup/lv_swap SYSFONT=latarcyrheb-sun16 crashkernel=auto rd_LVM_LV=VolGroup/lv_root  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM rhgb quiet console=ttyS0,115200n8 console=tty0   ---->console 설정 추가
        initrd /initramfs-2.6.32-754.35.1.el6.x86_64.img
title CentOS 6 (2.6.32-696.el6.x86_64)
        root (hd0,0)
        kernel /vmlinuz-2.6.32-696.el6.x86_64 ro root=/dev/mapper/VolGroup-lv_root rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD rd_LVM_LV=VolGroup/lv_swap SYSFONT=latarcyrheb-sun16 crashkernel=auto rd_LVM_LV=VolGroup/lv_root  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM rhgb quiet console=ttyS0,115200n8 console=tty0   ----> console 설정 추가
        initrd /initramfs-2.6.32-696.el6.x86_64.img
```
- upstart설정 (로그인 프롬프트 나오도록 설정)
	- centos6.9는 **SysV init가 아니라 Upstart**를 사용
- sysV init: 예전 리눅스 시스템에서 사용하던 초기화 방식
- upstart
	- `/etc/init/tty1.conf` 같은 잡(job) 파일 단위로 관리
	- centos6에서 사용
```
<파일 생성>
cat >/etc/init/ttyS0.conf <<'EOF'
# ttyS0 serial console (Upstart on CentOS 6)
start on stopped rc RUNLEVEL=[2345]
stop on runlevel [!2345]
respawn
exec /sbin/agetty -L 115200 ttyS0 vt100
EOF

<upstart 재로딩 및 즉시 시작>
[root@localhost init]# initctl reload-configuration
[root@localhost init]# start ttyS0

<적용 확인>
[root@localhost init]# ps -ef | grep [a]getty
root      1650     1  0 06:18 ttyS0    00:00:00 /sbin/agetty -L 115200 ttyS0 vt100

//[a]getty로 하면 grep agetty 매치 안됨
[a]getty=agetty와 동일하게 인식

<콘솔에서 root 로그인 허용>
[root@localhost init]# echo ttyS0 >> /etc/securetty
[root@localhost init]#
[root@localhost init]#
[root@localhost init]# cat /etc/securetty
console
vc/1
vc/2
vc/3
vc/4
vc/5
vc/6
vc/7
vc/8
vc/9
vc/10
vc/11
tty1
tty2
tty3
tty4
tty5
tty6
tty7
tty8
tty9
tty10
tty11
ttyS0   --->적용
```
- 확인
```
[root@localhost ~]# virsh console Centos6.9minimal
Connected to domain 'Centos6.9minimal'
Escape character is ^] (Ctrl + ])
Press any key to continue.
Press any key to continue.
Detected CPU family 6 model 106
Warning: Intel CPU model - this hardware has not undergone testing by Red Hat and might not be certified. Please consult https://hardware.redhat.com for certified hardware.
▒               Welcome to CentOS
Starting udev:                                             [  OK  ]
Setting hostname localhost.localdomain:                    [  OK  ]
Setting up Logical Volume Management:   3 logical volume(s) in volume group "VolGroup" now active
                                                           [  OK  ]
Checking filesystems
/dev/mapper/VolGroup-lv_root: clean, 26372/3276800 files, 622843/13107200 blocks
/dev/vda1: clean, 44/128016 files, 77485/512000 blocks
/dev/mapper/VolGroup-lv_home: clean, 11/2744320 files, 217264/10975232 blocks
                                                           [  OK  ]
Remounting root filesystem in read-write mode:             [  OK  ]
Mounting local filesystems:                                [  OK  ]
Enabling /etc/fstab swaps:                                 [  OK  ]
```
## 스냅샷 생성
- disk-only로 생성
- snapshot1_250917
```
[root@localhost ~]# virsh snapshot-create-as --domain Centos6.9minimal snapshot1_250917 "settingcomplete" --disk-only --atomic
Domain snapshot snapshot1_250917 created
[root@localhost ~]# virsh snapshot-list Centos6.9minimal
 Name               Creation Time               State
---------------------------------------------------------
 snapshot1_250917   2025-09-17 03:35:29 -0400   shutoff    <---스냅샷 찍을 때 당시 상태
 
 
[root@localhost home]# ls -l
total 77406288
-rw-r--r--.  1 qemu qemu    427819008 Mar 28  2017 CentOS-6.9-x86_64-minimal.iso
-rw-r--r--.  1 qemu qemu   1020264448 Jun 10 04:42 CentOS-7-x86_64-Minimal-2009.iso
-rw-------.  1 qemu qemu 107390828544 Sep 17 02:34 disk
-rw-------.  1 qemu qemu       851968 Sep 17 03:01 disk.snapshot1_250917
-rw-r--r--.  1 root root  53695545344 Sep 10 21:39 enpotest
-rw-r--r--.  1 root root         6852 Jul 18 08:05 enpotest.xml
-rw-------.  1 qemu qemu  64434601984 Sep 17 03:03 rocky8.4disk
-rw-r--r--.  1 qemu qemu   1980760064 Jun 20  2021 Rocky-8.4-x86_64-minimal.iso
-rw-r--r--.  1 qemu qemu  12851544064 May 29 20:06 Rocky-9.6-x86_64-dvd.iso
drwx------. 14 test test         4096 Jul 21 13:13 test
```
- 스냅샷 롤백 방법
	- `virsh snapshot-revert VM이름 스냅샷이름`
	- 가상머신 종료한 상태에서 진행
```
[root@localhost ~]# virsh snapshot-revert Centos6.9minimal snapshot1_250917
Domain snapshot snapshot1_250917 reverted
```
## 특이사항_virsh shutdown 안 됨
- centos6.9는 acpid가 없어서 virsh shutdown 신호를 받지 못함 (acpid 전원 버튼 신호)
```
yum install -y acpid
service acpid start
chkconfig acpid on
```
- qemu-guest-agent 설치
	- 게스트OS에서 동작하는 데몬 / kvm과 vm사이 통신 해주는 역할
	- 하이퍼바이저가 직접 게스트에게 종료 명령 내릴 수 있음
		- ACPI 처리 안되는 OS에 설치하면 좋음 (신규OS 설치 필요x)
```
[root@localhost ~]# yum -y install qemu-guest-agent
[root@localhost ~]# service qemu-ga start
Starting qemu-ga: [  OK  ]
[root@localhost ~]# chkconfig qemu-ga on
```