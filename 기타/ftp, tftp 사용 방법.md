# FTP (filezilla)
- filezilla > config > server listeners에서 protocol 설정 변경
	- plain FTP 사용
	- TLS 사용 시 오류 발생
- 스위치
```
bash-4.3# ftp
ftp> open
(to) 10.10.10.10
Connected to 10.10.10.10.
220-FileZilla Server 1.10.3
220 Please visit https://filezilla-project.org/
Name (10.10.10.10:admin): ftpuser
331 Please, specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
ftp> get
(remote-file) CentOS_181123.sh
(local-file) CentOS_181123.sh
local: CentOS_181123.sh remote: CentOS_181123.sh
200 PORT command successful.
150 Starting data transfer.
226 Operation successful
90474 bytes received in 0.0701 secs (1.3e+03 Kbytes/sec)
ftp> quit
```
- 전송 시 파일 깨지는 문제
	- type -> binary로 변경
```
ftp> type
Using ascii mode to transfer files.
ftp> binary
200 Type set to I
```
# tftp PLOS update 방법
- auto: 현재 설치 된 os의 반대편에 설치
- 주의 **펌웨어 다운로드 후 반드시 reload**
```
conf
os update tftp  <A.B.C.D> <FILE> [auto | os1 | os2] [forced]
os update ftp <A.B.C.D> <ID> <PASSWORD> <FILE> [auto | os1 | os2] [forced]

<중요>
reload
```