## 로그 확인
- 컨트롤러
```
[root@localhost log]# tail c-controller.log
2025-07-08 09:50:26.760  INFO 1148 [nioEventLoopGroup-4-2] c.p.c.b.c.DeviceConnectionManager       :47 - tcc connected. serial: CS2328GX21C2801426, remote: /192.168.212.249:34873
2025-07-08 09:50:26.781  INFO 1148 [nioEventLoopGroup-4-2] c.p.c.b.c.DeviceConnectionManager       :59 - tcc disconnected. serial: CS2328GX21C2801426, remote: /192.168.212.249:34873
```
- 스위치
	- 연결 실패 시에만 발생
```
bash-4.4# tail messages_etc
2025 Jul 08 09:50:27 KST (none) NSM[416]: [Notice][on_physical_uplink] Received signal from sysmgr uplink: , ticontroller: ge23, add_del 0
2025 Jul 08 09:50:27 KST (none) NSM[416]: [Notice][on_physical_uplink] Received signal from sysmgr uplink: ge23, ticontroller: ge15, add_del 1
```
## 해결
- 컨트롤러 ip가 바뀐 경우 9000,9001번 ip는 수동 변경 필요
	- `cd /install/cbroker.info`
```
bash-4.3# cat cbroker.info
[log]
status=disable
level=3
output=/var/log/cbroker.log
mute=true

[websocket]
ccc=wss://192.168.212.241:9000/websocket
connection_retry_time=5
connection_healthcheck_timeout=90
master=wss://192.168.212.244:9001/websocket   ----> 수동 변경 필요
backup=wss://192.168.212.244:9002/websocket
```
- 변경 후 cbroker 데몬 재시작 필요
	- 현재 cbroker 데몬 프로세스 번호 확인 후 `kill -9 pid`
	- `/bin/cbroker -m -d`명령어로 재시작
```
bash-4.3# ps -ef | grep cbroker
nobody     976     1  0 Jul07 ?        00:07:45 /bin/cbroker -m -d
nobody    5932  2337  0 04:32 ttyS0    00:00:00 grep cbroker

bash-4.3# kill -9 976

bash-4.3# /bin/cbroker -m -d
```