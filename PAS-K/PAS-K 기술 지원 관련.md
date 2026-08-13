# 로그
- `/opt/k2/var/log`
	- AEU가 생성하는 로그
- `/var/lib/tftpboot`
	- SFU가 생성하는 로그
	- 그 외 로그는 모두 aeu가 생성한 것

# 모델
k3
앞자리 홀수
8620 예외

k2
앞자리 짝수

k4 - 신모델

# SFR(SFU Fast Recovery)
SFR 발생 이유?
K2에서만 발생하는 이유
K3에서는 발생하지 않음

SFR 재기동 하면 원래는 SFU OS 재부팅이 됨

SWR
->SFU 일부 데몬만 재기동, 서비스에는 영향 없음
v2.2.4 이상부터 생긴 기능

SWR 기능을 사용하기 위해서는
CPLD (메인 보드bios 관련 펌웨어) 업데이트 필요

OS는 최신인데 CPLD가 작업이 안되어 있는 경우 SWR 안된다는 로그가 찍힘

K3에서 sfu 재기동 watchdog 발생하면 별도의 분석 필요.

SFU 발생 시 봐야 하는 것
/opt/k2/var/log/syslog, /opt/k2/var/log/k2/amss.keepsfu.log (aeu)
messages (sfu)