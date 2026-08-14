# 로그
- `/opt/k2/var/log`
	- AEU가 생성하는 로그
- `/var/lib/tftpboot`
	- SFU가 생성하는 로그
	- 그 외 로그는 모두 aeu가 생성한 것

- /opt/k2/var/log/syslog : system 동작에 대한 전반적인 이벤트가 포함되어 있는 로그, 로그 분석 시 1순위로 보게 됨
	- 원래는 데몬별로 따로 로그 남김, syslog는 전부 합쳐서 출력

```
- /opt/k2/var/log/messages : 실제 장비에서 show log로 확인이 되는 로그 정보, syslog에 비해 생략된 부분이 많음
- /opt/k2/var/log/clicmds : 사용자가 CLI 에서 커맨드 실행한 내역이 기록된 로그, 로그뒤에 보이는 초단위 시간은 해당 명령어가 실행되기까지의 시간을 의미
- /opt/k2/var/log/k2/amss.keepsfu.log : AEU -> SFU에 대한 H/C 상태 및 Fast Recovery 이벤트가 기록된 로그
- /opt/k2/var/log/k2/amss.statistics.log : 1분단위로 통계데몬 실행 결과가 기록된 로그, 해당 로그 확인으로 AEU 정상 동작 여부 확인 가능
	- pask aeu 자체적으로 실행하는 통계 데몬. (게시판 통계 등)
- /opt/k2/var/log/k2/amss.log : 설정 적용 시 설정 데몬에 대한 실행 결과가 기록된 로그, 설정 오류 등이 난다면 해당 로그에서 상세 확인이 가능

- /opt/k2/var/log/k2/amss.firmware.log : OS 업데이트 이벤트에 대한 진행 사항이 기록된 로그, 보안적합성 모드라서 상세 OS 버전 확인이 불가할 경우 해당 로그에서 확인이 가능
	- pask는 보안적합성모드, 일반모드 두 가지 지원
```
# 모델
k3 :앞자리 홀수
8620 예외

k2 : 앞자리 짝수

k4 :신모델

# SFR(SFU Fast Recovery)
- AEU는 SFU를 icmp- telnet 순서로 실시간 모니터링
- SFU가 응답이 없을 경우 SFU를 재기동하게 됨 (SFR 동작)

- SFR 발생 이유?
	- K2에서만 발생하는 이유 
	- K3에서는 발생하지 않음
	- K2에서 발생 원인은 확인되지 않음
- SFR 원인 확인
	- `/var/lib/tftpboot/sfu_log/sfu_backup/messages_etc`
```
Aug 11 13:24:54 UTC (none) kernel: reboot reason: Watchdog

AEU가 sfu에 대한 헬스체크가 실패해서 sfu 재기동 시킨 동작이 watchdog
```

- SFR 재기동 하면 원래는 SFU OS 재부팅이 됨, 따라서 이것을 방지하고자 추후 SWR 기능 개발

SWR
->SFU 일부 데몬만 재기동, 서비스에는 영향 없음
v2.2.4 이상부터 생긴 기능

SWR 기능을 사용하기 위해서는
CPLD (메인 보드bios 관련 펌웨어) 업데이트 필요
OS는 최신인데 CPLD가 작업이 안되어 있는 경우 SWR 안된다는 로그가 찍힘

K3에서 sfu 재기동 watchdog 발생하면 별도의 분석 필요.

SFU 발생 시 봐야 하는 것
- /opt/k2/var/log/syslog, /opt/k2/var/log/k2/amss.keepsfu.log (aeu)
- messages (sfu)



# 기타
- SFU와 AEU는 OS, H/W 모두 다르게 사용
- SFU가 부팅 중 cpu 부하가 일시적으로 많이 찰 수 있다 (문제인지 보려면 지속적인지 확인해야 함)
- `/var/lib/tftpboot/messages_all` 전체 부팅 로그
```
Aug 11 22:25:16 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge22")
Aug 11 22:25:17 KST (none) hwmon: [K2-M04/SYS:HISTORY] Event alarm generate (type="CPU",threshold="80 %",used_rate="93 %")
Aug 11 22:25:17 KST (none) login[1258]: [K2-M04/USER:HISTORY] Log In (user="root", from="pts/2")
Aug 11 22:25:17 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge18")
Aug 11 22:25:18 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge23")
Aug 11 22:25:18 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge19")
Aug 11 22:25:19 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge24"
```